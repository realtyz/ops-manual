# GPU 与网卡的 PCIe 可达性和显存注册条件

很多团队在上线高速集群时会遇到同一个头疼场景：主机内存 RDMA 跑出了漂亮的线速，单卡 CUDA 训练和推理也都一切正常，可只要开启 GPUDirect RDMA，程序要么在显存注册时直接报错，要么性能断崖式下跌、甚至悄悄回退到主机内存中转。

要让传统 PCIe 路径上的 GPUDirect RDMA 真正跑通，必须同时闯过两道关卡：**硬件与路由层的 PCIe 可达性**（物理拓扑、ACS、IOMMU），以及**软件驱动层的显存注册能力**（DMA-BUF 或 Peer Memory）。排查时永远遵循一个铁律：先锁定实际配对的那一张 GPU 和那一张网卡，再顺着拓扑树和内核状态逐关验票。

## 第一关：把 GPU 与网卡还原到同一棵 PCIe 树

不要凭直觉猜“设备都在 NUMA 0 就一定直连”。在 `node-1` 上，先把应用实际用到的 CUDA 设备（注意 `CUDA_VISIBLE_DEVICES` 会改写进程内编号）与 RDMA 设备各自还原成 PCI BDF，再通过 `lspci -t` 找到它们的共同上游：

```bash
# 1. 查询物理 GPU 的序号、UUID 与 PCI BDF
$ nvidia-smi --query-gpu=index,uuid,pci.bus_id --format=csv
index, uuid, pci.bus_id
0, GPU-11223344-5566-7788-99aa-bbccddeeff00, 00000000:82:00.0

# 2. 查出 RDMA 设备 mlx5_0 对应的 PCI BDF（与 04 篇一致）
$ readlink -f /sys/class/infiniband/mlx5_0/device
/sys/devices/pci0000:80/0000:80:01.1/0000:81:00.0/0000:81:02.0/0000:83:00.0

# 3. 对照拓扑树：确认 GPU 0 与 mlx5_0 挂在同一个 PCIe Switch 下
PCIe Root Complex（省略中间桥）
└── PCIe Switch（上游桥 0000:81:00.0）
    ├── 下游桥 0000:81:01.0
    │   └── GPU 0      0000:82:00.0
    └── 下游桥 0000:81:02.0
        └── mlx5_0/1   0000:83:00.0
```

配合第 04 章学过的 `nvidia-smi topo -m`，可以迅速判断 GPU 与网卡的物理距离：
- **`PIX`（最优）**：只经过单个 PCIe Switch，两设备同处一颗交换芯片下，天然具备芯片内直接转发的物理基础；
- **`PXB`（次优）**：跨越多个 PCIe Switch，但尚未经过 CPU 的 PCIe 主桥；
- **`PHB`（谨慎核查）**：需要经过同一颗 CPU 的 PCIe Host Bridge，必须确认主板与芯片组支持主桥对等转发，且往往带宽/延迟逊于 `PIX`；
- **`SYS`（传统路径应避开）**：跨越处理器间互联（跨 Socket/NUMA 节点），传统 PCIe GPUDirect RDMA 要求两设备共享上游 Root Complex，跨 Socket 往往性能骤降或根本不受支持。

> [!warning] 常见误区：同属一个 NUMA 节点就等于 `PIX`
> 现代多路服务器一颗 CPU 往往引出多个独立的 PCIe Root Port。两张设备即使都归属 NUMA 0，如果分别插在不同根端口下，它们在拓扑矩阵里会呈现 `PHB` 而非 `PIX`，流量必须绕道 CPU 根复合体。

## 第二关：ACS 决定转发方向，IOMMU 决定 DMA 地址映射

哪怕 `nvidia-smi topo -m` 已经给出了完美的 `PIX`，PCIe 桥和主板固件里的两个硬件开关仍可能把近在咫尺的直达流量“拐走”甚至拦死。

### 1. ACS（Access Control Services，访问控制服务）

**ACS** 是 PCIe 协议层的设备访问控制机制，用来决定下游设备发出的对等事务能不能在 PCIe Switch 内部直接转给邻居。其中最关键的两个控制位是**请求重定向（`ReqRedir`）** 和**完成重定向（`CmpltRedir`）**：一旦开启，下游桥就会把对等请求强制发往上游 Root Complex 接受审查，而不是在交换芯片内就地抄近路。

以发起端网卡读取本地 GPU 显存为例，存在一去一回两条 PCIe 事务：先是网卡向 GPU 发出**读请求**，随后 GPU 把带着显存数据的 **PCIe 完成包**发回网卡（图 06-05-01 中的载荷箭头对应后一方向）。用 `lspci -vv` 检查网卡侧下游桥 `0000:81:02.0` 的 `ACSCap`（硬件能力）和 `ACSCtl`（当前生效控制位）：

```bash
$ lspci -vv -s 0000:81:02.0 | grep -A 2 "Access Control Services"
        Capabilities: [220 v1] Access Control Services
                ACSCap: SrcValid+ TransBlk+ ReqRedir+ CmpltRedir+ UpstreamFwd+ EgressCtrl- DirectTrans-
                ACSCtl: SrcValid+ TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
```

哪怕拓扑距离同为 `PIX`，下游桥上仅仅多开一个 `ReqRedir+`，路径命运就可能完全不同：

| 观察场景 | 拓扑距离 | 网卡侧下游桥 `ACSCtl` 读数 | 路径影响与下一步动作 |
| --- | --- | --- | --- |
| **场景甲（放行直达）** | `PIX` | `SrcValid+ ReqRedir- CmpltRedir-` | 该桥已关闭对等重定向；继续核对 GPU 侧下游桥与其余相关桥。 |
| **场景乙（被拽向上游）** | `PIX` | `SrcValid+ ReqRedir+ CmpltRedir-` | 网卡发往 GPU 的读请求会被强行重定向至上游 Root Complex；需检查上游是否支持转回以及是否触发性能回退。 |

裸机 AI 计算节点通常会按照服务器硬件指南关闭 GPU 与网卡之间 PCIe 桥的 ACS（或单独关闭 `ReqRedir` / `CmpltRedir`），让读写事务全程留在 PCIe Switch 内部高速折返。但如果是**虚拟化或 VFIO 设备直通**环境，虚拟机隔离高度依赖 ACS 防止跨租户窃听，切不可盲目一刀切关闭。

### 2. IOMMU（Input–Output Memory Management Unit，输入输出内存管理单元）

**IOMMU** 位于设备与内存之间，负责为外设 DMA 提供地址转换和隔离保护。根据 CUDA 13.2.1 GPUDirect RDMA 指南，传统 PCIe 路径要求参与直达的设备看到一致的物理地址；因此启用 IOMMU 时，必须确认相关设备处于直通一一映射（`identity` / `passthrough`）模式，否则硬件地址转换会阻断对等访问。

现场排查时，除了检查 BIOS 设置和 `/proc/cmdline` 启动参数，更直接的证据在 sysfs 中：

```bash
# 查看网卡 BDF 所属的 IOMMU 分组，以及该分组的默认映射域类型
$ readlink -f /sys/bus/pci/devices/0000:83:00.0/iommu_group
/sys/kernel/iommu_groups/42

$ cat /sys/kernel/iommu_groups/42/type
identity    # identity 表示直通一一映射；若为 DMA / DMA-FQ 则经过地址转换
```

> [!note] 新架构平台边界
> 面向虚拟化直通的特殊配置，以及新一代 Grace Blackwell 搭配 ConnectX-8 **Data Direct** 等全新互连架构，其 PCIe 拓扑与 IOMMU 约束与传统 x86 PCIe 裸机路径存在差异，部署时应直接对照对应平台的专项配置指南。

## 第三关：DMA-BUF 导出导入链路与 Peer Memory 对比

物理路径和开关全部绿灯后，最后一步落在软件栈能否真正为一块显存签发 MR。当前 Linux 6.8+ 主线内核与 CUDA 13.2.1 栈中，**DMA-BUF** 是官方优先推荐的标准方案，整个注册过程分三步推进：

```c
// 步骤 1：查询目标 GPU 是否支持 DMA-BUF 显存导出
cuDeviceGetAttribute(&supported, CU_DEVICE_ATTRIBUTE_DMA_BUF_SUPPORTED, dev);

// 步骤 2：在 GPU 上分配显存并导出为 Linux DMA-BUF 文件描述符（fd）
// 约束：地址与长度必须按主机页大小对齐；若用虚拟地址预留，全区间须完整映射且归属同一设备与属性
cuMemAlloc(&dptr, size);
cuMemGetHandleForAddressRange(&dmabuf_fd, dptr, size, CU_MEM_RANGE_HANDLE_TYPE_DMA_BUF_FD, 0);

// 步骤 3：把 fd 导入本地 RDMA 保护域（PD），注册生成可供远程读写的显存 MR
// 约束：MR 基址 iova 的页内偏移必须与文件偏移 offset 保持一致
mr = ibv_reg_dmabuf_mr(pd, offset, size, iova, dmabuf_fd,
                       IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_WRITE | IBV_ACCESS_REMOTE_READ);
```

注意一个关键细节：导出的文件描述符 `dmabuf_fd` 只服务于本机跨驱动交接。一旦 `ibv_reg_dmabuf_mr()` 成功返回 MR，跨机传输时发起端依然只使用远端发布的 RDMA 地址、长度与 `rkey`，网络协议层面完全不需要传递文件描述符。

这三步恰好天然切出了三条精准的排障分界线：
- **导出成功**：说明 GPU 驱动接受了该块显存分配的 DMA-BUF 导出条件；
- **导入成功**：进一步说明网卡内核驱动、用户态 provider 与固件接受了这组映射参数；
- **实际请求成功**：只有真实跑通一次工作请求并收到 `IBV_WC_SUCCESS`，才构成该 GPU–网卡配对在该方向上支持 GPUDirect RDMA 的闭环证据。

至于传统的 **Peer Memory** 路径，它依靠 `nvidia-peermem` 内核模块（早期旧系统称 `nv_peer_mem`）把显存映射回调挂接到 OFED 栈上。排查旧环境时尤其要核实两点：

- **模块在跑 ≠ 流量真走它**：`lsmod` 看到 `nvidia-peermem` 已加载只能证明内核组件存在；现代 NCCL 等通信库在环境支持时会优先选用主线 DMA-BUF。
- **安装顺序与接口匹配**：若 GPU 驱动安装早于 MLNX_OFED，必须确保 `nvidia-peermem` 按目标 RDMA 内核 API 重新构建，否则符号不匹配会导致显存注册静默失效。

> [!note] 统一软件栈与 Kubernetes 集群前提
> 对齐全书参考栈（**Ubuntu 24.04 LTS / Linux 6.8+、NVIDIA 开源内核模块 R595、CUDA 13.2.1**），Turing 及更新架构 GPU 可直接通过主线 **DMA-BUF** 与受支持的发行版或厂商网络驱动配合，彻底告别逐台编译维护 `nvidia-peermem` 的烦恼。
>
> 在 Kubernetes 集群中，**GPU Operator（v26.7.2）** 官方优先推荐原生 DMA-BUF 路径，并与 **Network Operator** 协同完成 Pod 高速网络交付（详见第 12–13 章）。

> [!tip] 推理集群 PD 分离上线前核查
> 部署基于 NIXL 或 vLLM `NixlConnector` 的 PD 分离推理资源池前，务必按本篇顺序把每台 Prefill 与 Decode 节点的 `PIX` 拓扑、`ACSCtl`、`iommu_group/type` 与 DMA-BUF 注册全部过一遍。一旦某一关失守导致底层静默回退到主机内存中转或 TCP，长上下文请求的 TTFT 就会在上线后突发暴涨。

软硬件条件都对照清楚了，怎么用最小的隔离实验把“主机内存 RDMA 正常”和“GPUDirect RDMA 正常”分别验实？下一篇，我们用 `perftest`（`ib_write_bw`）做一轮干净的两阶段对照测量。

> [!info]- 参考资料
> - **CUDA 13.2.1 GPUDirect RDMA 与显存导出接口**：[Supported Systems (PCIe / IOMMU)](https://docs.nvidia.com/cuda/archive/13.2.1/gpudirect-rdma/index.html#supported-systems)、[Using `nvidia-peermem`](https://docs.nvidia.com/cuda/archive/13.2.1/gpudirect-rdma/index.html#using-nvidia-peermem)、[CUDA Memory Management (`group__CUDA__MEM`)](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-driver-api/group__CUDA__MEM.html)、[CUDA Device Management (`group__CUDA__DEVICE`)](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-driver-api/group__CUDA__DEVICE.html)
> - **PCIe P2P、ACS 与 IOMMU 内核与工具文档**：[NVIDIA SMI Documentation](https://docs.nvidia.com/deploy/nvidia-smi/)、[Linux Kernel PCI P2P DMA](https://docs.kernel.org/driver-api/pci/p2pdma.html)、[pciutils v3.13.0 `ls-ecaps.c`](https://github.com/pciutils/pciutils/blob/v3.13.0/ls-ecaps.c)、[NCCL 2.27.3 ACS Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html#pci-access-control-services-acs)、[Linux Kernel VFIO](https://docs.kernel.org/driver-api/vfio.html)、[Linux v6.12 `sysfs-kernel-iommu_groups`](https://github.com/torvalds/linux/blob/v6.12/Documentation/ABI/testing/sysfs-kernel-iommu_groups)
> - **DMA-BUF 注册、GPU Operator 与新平台参考**：[rdma-core v57.0 `ibv_reg_mr(3)`](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/man/ibv_reg_mr.3)、[NVIDIA GPU Operator GPUDirect RDMA & GDS Guide](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-rdma.html)、[NVIDIA GPU Operator Platform Support](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html)、[Grace Blackwell ConnectX-8 Data Direct Guide](https://docs.nvidia.com/multi-node-nvlink-systems/grace-blackwell-cx8-gpudirect-rdma-guide/platform_software_and_configuration.html)
