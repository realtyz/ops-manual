# GPUDirect Storage 的直达路径与验证

前面四篇里，数据从 NVMe 进显存总要在主机内存里倒一手。字节先进内核页缓存，再进应用缓冲，最后才走 H2D 过一遍 PCIe。盘和 GPU 明明挂在同一台服务器的 PCIe Switch 下，为什么非要多绕主机内存一圈？

**GPUDirect Storage（GDS）** 就是为了裁掉这段主机折返跑而生的。应用通过 **cuFile** 接口递交文件句柄、偏移、长度与目标显存地址。符合条件的存储控制器即可经 PCIe 点对点 DMA，把文件载荷直接写进 GPU 显存。

为了纯看搬运路径本身，本篇切到一个干净的单机场景。本地 NVMe 上存放着已经预处理好的定长张量记录，读进显存就能直接算（在线推理用 `fastsafetensors` 直读权重、或从本地 NVMe 换入 KV 块也是典型落地场景）。如果数据仍是需要 CPU 解码的压缩图像，单换读取接口无济于事。

## 一、普通缓冲读、GDS 直达与兼容路径

直达的本质是**载荷数据绕开了主机内存与应用 H2D**。但控制流依然归 CPU 管：打开文件、注册句柄、下发 `cuFileRead` 与回收完成状态，全都要靠主机线程发起。

![三栏使用相同的单机拓扑。左栏中，载荷经页缓存和应用缓冲进入显存。中栏中，NVMe 经 PCIe Switch 将载荷直接送到显存缓冲。右栏中，载荷经主机缓冲进入显存。蓝色虚线表示三种路径都由 CPU 提交 I/O。](images/fig-09-05-01-gds-read-paths.png)

*图 09-05-01：同一台节点上的三种 NVMe 读取路径。橙色实线为数据载荷，蓝色虚线为 CPU 发起的 I/O 控制。中栏直达要求设备拓扑、文件请求与目标显存缓冲同时满足后端条件。*

把图里的三条路径拆成一张对照表，差异一目了然：

| 读取路径 | 数据载荷流经的物理站 | 是否经主机数据缓冲与 H2D | 核心工程判断点 |
| --- | --- | --- | --- |
| **普通缓冲读取** | NVMe → 页缓存 → 应用缓冲 → 锁页缓冲 → GPU | 是 | 热页缓存能省读盘，但主机拷贝、CPU 消耗与 H2D 照跑 |
| **GDS 直达** | NVMe → PCIe P2P DMA → GPU 显存缓冲 | 否（绕开主机内存） | 省下主机内存带宽与 CPU 搬运，但严苛依赖软硬件条件 |
| **cuFile 兼容路径** | cuFile 借道 POSIX/主机缓冲取回数据，再 H2D 入显存 | 是（自动降级回退） | 保证程序不报错中断，但吞吐与开销退回主机中转模式 |

底层直达路径还有两处容易看混的细节：
- **显存内部中转 ≠ 主机中转**：若未提前用 `cuFileBufRegister` 注册复用显存缓冲，或单次请求对齐不佳，cuFile 可能先 DMA 到一块内部 GPU 暂存区，再在卡内拷到目标地址。这仍属于设备内部搬运，和退回主机内存截然不同。
- **远端 GDS 与 NCCL GDR 各管各的**：受支持的并行文件系统或 RDMA 存储同样支持远端 GDS，由网卡把远端载荷直接送进显存。但第 06 章的 GPUDirect RDMA 解决的是网卡通信，NCCL 日志里的 GDR 标志只覆盖集合通信，无法替代 cuFile 对文件 I/O 的核验。本机 NVMe 走 GDS 更无需额外加装 RDMA 网卡。

## 二、决定请求能否走直达的三道门槛

调用了 `cuFileRead` 且顺利拿到数据，距离“真正走了 GDS 直达”还差着三道硬门槛。

### 1. 文件句柄与 `O_DIRECT` 标志

应用先打开目标文件，再用 `cuFileHandleRegister` 注册句柄，随后提交偏移、长度与 GPU 缓冲地址。

很多人习惯把 `O_DIRECT` 当成 GDS 开关，其实混淆了两层语义。`O_DIRECT` 只表达“跳过主机页缓存的直接 I/O”，并不指定目的地是显存。反过来，从 CUDA 12.2（GDS 1.7）起，cuFile 也接受未带 `O_DIRECT` 打开的文件描述符。只要直达路径可用，请求照样走 GDS；不可用且允许兼容模式时，再退回 POSIX 路径。

### 2. 偏移、长度与显存缓冲对齐

直达后端对对齐极为敏感。工程上通常准备 **4 KiB 对齐的 GPU 缓冲**，按 **4 KiB 整数倍**设置文件与缓冲偏移，并整块读取连续记录（例如单次读 `1 MiB`）。对复用的显存缓冲，建议显式调用 `cuFileBufRegister` 避免内部中转。一旦出现非对齐切片或后端不支持该文件系统，cuFile 便可能静默降级到兼容路径，甚至直接报错。

### 3. 硬件拓扑与上游 PCI P2PDMA 后端

早期本地 NVMe 走 GDS 强依赖内核模块 `nvidia-fs` 与驱动补丁，内核升级维护成本很高。从 CUDA 12.8（GDS 1.13）起，Linux x86_64 平台引入了基于主线内核 **PCI P2PDMA** 的上游直达路径。单盘 NVMe 可直接跳过 `nvidia-fs` 补丁实现卡盘直通。

> [!note] 版本与平台说明：对齐 `CUDA 13.2.1` 与上游 PCI P2PDMA
> 按本书统一基线（Ubuntu 24.04 LTS、Linux 6.8+、OpenRM R595、`CUDA 13.2.1` 内置 `cuFile 1.17.0.44`），单盘 XFS 本地 NVMe 已原生满足上游 PCI P2PDMA 的软件条件。
>
> 历史上不同官方指南对最低驱动口径略有差异（架构文档记为 OpenRM R570+，部署指南记为 R565.47+；RHEL 9.4 的 5.14 内核亦通过回移植支持）。若涉及多盘软 RAID、NVMe 多路径或远端并行存储，须查具体存储支持矩阵。

## 三、配置开关与三层取证闭环

开启兼容回退（`allow_compat_mode: true`）后，直达条件哪怕没满足，程序也会悄悄退回主机拷贝照常跑完。因此生产验收绝不能只看“读出数据没报错”，必须把配置开关和三层取证闭环起来。

启用上游本地 NVMe P2PDMA 时，须在 `/etc/cufile.json` 的**全局属性和 NVMe 块设备属性两处**同时打开 `use_pci_p2pdma`：

```json
{
  "properties": {
    "allow_compat_mode": true,
    "use_pci_p2pdma": true
  },
  "block": {
    "nvme": {
      "use_pci_p2pdma": true
    }
  },
  "profile": {
    "cufile_stats": 3
  }
}
```

配置生效后，按“平台候选能力 → 进程实走路径 → 同条件 A/B 对比”三层逐级取证：

```bash
# 第一层：核对挂载、PCIe 层级与平台支持的候选模式
findmnt -T /mnt/nvme/records.bin      # 确认目标文件确实落在本地 XFS/NVMe 挂载上
lspci -t                              # 确认 NVMe 与目标 GPU 之间的 PCIe 拓扑可达
gdscheck -p                           # 查看候选后端；看到 "NVMe: p2pdma, compat" 仅说明具备候选能力

# 第二层：采集活动进程实时统计，核实本次请求到底有没有静默降级
gds_stats -p <pid> -l 3               # 重点盯每卡总请求 n、兼容降级计数 posix、非对齐 unalign 与 MiB
# 若怀疑静默回退，在短时诊断轮打开 logging.level=TRACE（切勿带入性能压测）：
# 日志出现 "cufile IO mode: POSIX" 即铁证已退回兼容路径

# 第三层：对照直达对比基准，用环境变量强制回退做同条件 A/B 测速
CUFILE_FORCE_COMPAT_MODE=true ./read_records_bench
```

> [!warning] 常见误区：`/proc/driver/nvidia-fs/stats` 没增长 ≠ 没走 GDS
> `/proc/driver/nvidia-fs/stats` 只统计经过传统 `nvidia-fs` 模块的请求。采用主线内核 PCI P2PDMA 时，请求根本不经过 `nvidia-fs`。该文件计数为零，既可能是成功走了上游 P2PDMA，也可能是退回了兼容路径。必须结合 `gds_stats` 里的 `posix` 计数、TRACE 日志与 P2PDMA 专属统计交叉核验。

## 四、局部 I/O 快了，整步就一定快吗

做 GDS 收益对比时，务必固定文件、偏移、请求大小、缓冲、并发度与缓存冷热状态。先跑确认走 P2PDMA 的直达组，再用 `CUFILE_FORCE_COMPAT_MODE=true` 跑同条件兼容组。普通缓冲读的热页缓存基线则应单独列出，切忌把“直达 vs 兼容”和“冷读 vs 热读”混在一起测。

就算确认单次读取比兼容路径快了 **6 ms**，端到端训练步时也未必缩短。假如这次读取原本藏在前一批 `125 ms` 的计算期间，且预取队列从未见底，提前 `6 ms` 交货只是让数据在队列里多等 `6 ms`。训练步时依然稳稳停在 `125 ms`。

> [!summary] 核心结论：什么时候 GDS 真正值回票价
> 1. **I/O 站在关键路径上**：例如在线推理扩容时加载百 GB `safetensors` 权重（`fastsafetensors`）、训练故障恢复读大块 Checkpoint，或从本地 NVMe 突发换入长上下文 KV 缓存。此时没有长计算可掩盖读取，省下的每一毫秒都直接缩短启动 RTO 或首字延迟。
> 2. **主机内存带宽或 CPU 告急**：即便端到端步时未变，绕开主机内存也能释放大量 CPU 周期与 DRAM 带宽，缓解多卡争用。反之，若样本仍须 CPU 解码，就必须先把解码迁往 GPU（如 DALI）或提前预处理，单换 API 无法加速整条管线。

单机局部直达讲清了。但在多机训练集群里，慢步往往伴随存储抖动、CPU 抢占、H2D 阻塞和集合通信拉长同时出现。下一篇我们站回多 Rank 全局时间线，学一套百试百灵的逐段隔离排障法。

> [!info]- 参考资料
> - **GPUDirect Storage 架构与最佳实践**：[NVIDIA GDS Overview Guide](https://docs.nvidia.com/gpudirect-storage/overview-guide/index.html)、[GDS Software Architecture](https://docs.nvidia.com/gpudirect-storage/overview-guide/software-architecture.html)、[GDS Best Practices Guide (PDF)](https://docs.nvidia.com/gpudirect-storage/pdf/best-practices-guide.pdf)、[GDS O_DIRECT Guide](https://docs.nvidia.com/gpudirect-storage/o-direct-guide/index.html)、[cuFile API Reference](https://docs.nvidia.com/gpudirect-storage/api-reference-guide/)
> - **上游 PCI P2PDMA 部署、版本与发行说明**：[CUDA 13.2 Release Notes](https://docs.nvidia.com/cuda/archive/13.2.0/release-notes/index.html)、[GDS Release Notes · New Features & Changes](https://docs.nvidia.com/gpudirect-storage/release-notes/new-features-and-changes.html)、[GDS Release Notes (PDF)](https://docs.nvidia.com/gpudirect-storage/pdf/release-notes.pdf)、[NVMe P2PDMA Pre-Installation Steps](https://docs.nvidia.com/deploy/gds-support-for-nvme/latest/pre-installation-steps.html)、[GDS Configuration Guide](https://docs.nvidia.com/gpudirect-storage/configuration-guide/index.html)
> - **配置、统计与故障诊断**：[NVMe P2PDMA `cufile.json` Settings](https://docs.nvidia.com/gpudirect-storage/troubleshooting-guide/nvme-p2pdma-troubleshooting.html#cufile-json-settings)、[P2PDMA Statistics](https://docs.nvidia.com/gpudirect-storage/troubleshooting-guide/nvme-p2pdma-troubleshooting.html#statistics)、[GDS & Compatibility Mode](https://docs.nvidia.com/gpudirect-storage/troubleshooting-guide/advanced-troubleshooting.html#gds-and-compatibility-mode)、[`gds_stats` Library Tracing](https://docs.nvidia.com/gpudirect-storage/troubleshooting-guide/gds-library-tracing.html#example-displaying-sample-user-level-statistics-for-each-gds-process)、[User-Space Counters per GPU](https://docs.nvidia.com/gpudirect-storage/troubleshooting-guide/user-space-counters-in-gpudirect-storage.html#distribution-of-io-usage-in-each-gpu)、[Displaying `nvidia-fs` Driver Statistics](https://docs.nvidia.com/gpudirect-storage/troubleshooting-guide/displaying-gds-nvidia-fs-driver-statistics.html)、[GDS Environment Variables](https://docs.nvidia.com/gpudirect-storage/troubleshooting-guide/installing-gpudirect-storage.html#environment-variables-used-by-gpudirect-storage)
