# 传输吞吐、GPU 放置与综合检验

把一个双卡任务部署到四卡 PCIe 服务器上时，工程师经常面临一个经典两难：
- **选同 Switch 两张卡（如 `GPU 0 + GPU 1`）**：卡间可以走 Switch 内部 P2P 直达，但两张卡同时从主机拉输入时要挤同一条上行；
- **选跨 Switch 两张卡（如 `GPU 0 + GPU 2`）**：两条独立上行让主机输入吞吐直接翻倍，但卡间交换被迫绕道主机内存并跨 Socket 中转。

到底哪种放卡更快？答案不在直觉里，而在**逐阶段记账与分界点计算**里。本篇先统一吞吐测量口径与放置权衡模型，再拆解 `nvbandwidth v0.11.0` 的真实读法，最后用四道无实机综合实战题闭合全章知识。

## 单路与并发吞吐的统一测量口径

做任何多卡带宽对比前，先把测量边界锁死，否则两组数字根本不可比：

1. **身份与路径固定**：登记实际物理 GPU 的 UUID、BDF、进程序号，明确源/目的端、有效载荷方向、字节数、复制方式（CE 或 SM），以及主机缓冲区是否页锁定和真实物理页落点。
2. **剔除冷启动开销**：设备初始化、显存分配、锁页注册与预热拷贝全部放在计时窗口之外。
3. **对齐并发终表时刻**：测量并发吞吐时，须用同一台主机的高精度时钟计时，等待所有参与卡的相关 Stream 全部同步完成并校验目的数据后停表；记录中同时保留各路单路耗时与整体共同完成时间。

设共交付有效字节 $V_{\text{total}}$，从共同起点到最后一路完成耗时为 $T_{\text{joint}}$，则单路吞吐与并发总吞吐统一按以下口径计算：

$$B_{\text{single}} = \frac{V_i}{T_i}, \qquad B_{\text{agg}} = \frac{\sum_i V_i}{T_{\text{joint}}}$$

在健康的双路四卡 Gen4 x16 节点上（每段每方向有效预算 `25 GB/s`，跨 Socket 远端 H2D 整条路径有效速率 `20 GB/s`，缓冲均已锁页，每路交付 `1 GB` 且通过数据校验），五组基准对照记录如下：

| 传输组合（每路 `1 GB`） | 实际路径与并发关系 | 共同完成时间 | 有效吞吐 |
| --- | --- | ---: | ---: |
| `M0 → GPU 0`（或 `M0 → GPU 1` / `M1 → GPU 2`） | 三次独立本地 H2D 单测 | 各 `40 ms` | 各 `25.00 GB/s` |
| `M0 → GPU 0` 与 `M0 → GPU 1` | 并发本地 H2D，同向均分上行 A | `80 ms` | 各 `12.50 GB/s`，合计 `25.00 GB/s` |
| `M0 → GPU 0` 与 `M1 → GPU 2` | 并发本地 H2D，分走独立上行 A、B | `40 ms` | 各 `25.00 GB/s`，合计 `50.00 GB/s` |
| `GPU 0 → GPU 1` | Switch A 内部 PCIe P2P 直达 | `40 ms` | `25.00 GB/s` |
| `GPU 0 → GPU 2` | 两段串行中转：本地 D2H 写 `M0` → 跨 Socket H2D 读 `M0` | `90 ms` | `≈ 11.11 GB/s` |

对照这组数字，重点看清两个换算细节：
- **同分支 vs 跨分支并发输入**：同分支上行 A 在 `0.08 s` 内同向交付 `2 GB`，总吞吐为 $2 \div 0.08 = 25\ \text{GB/s}$；换用两条独立上行后，同样 `2 GB` 在 `0.04 s` 内到达，总吞吐翻倍至 $50\ \text{GB/s}$。
- **跨分支串行主机中转**：第一段本地 D2H 用时 $1 / 25 = 0.04\ \text{s}$，第二段跨 Socket H2D 用时 $1 / 20 = 0.05\ \text{s}$，完整中转耗时与最终有效吞吐为：

$$T_{\text{cross-xchg}} = \frac{1}{25} + \frac{1}{20} = 0.09\ \text{s}\ (90\ \text{ms}), \qquad B_{\text{eff}} = \frac{1\ \text{GB}}{0.09\ \text{s}} \approx 11.11\ \text{GB/s}$$

注意：拓扑标签 `SYS` 只说明路径穿过了跨 NUMA 系统互联，并不自带固定的速率数字；线上遇到类似退化时，仍须结合负载下 `LnkSta`、缓冲区物理页位置与链路/内存流量区分究竟是上行饱和还是内存控制器争用。

## 主机输入量与卡间交换量决定放置分界

现在把“输入”和“卡间交换”放进同一轮作业：每轮先由两张卡并发接收各 `1 GB` 本地输入，两路输入全部完成后，发送卡再向接收卡交付 $S\ \text{GB}$ 中间结果（忽略两种放置相同的计算耗时与固定提交开销，跨分支采用经 `M0` 串行整块中转）。

分别写出两种放置每轮的数据搬运总耗时：

$$T_{\text{同分支}}(S) = 80 + 1000 \times \frac{S}{25} = 80 + 40S\ (\text{ms})$$

$$T_{\text{跨分支}}(S) = 40 + 1000 \times \left(\frac{S}{25} + \frac{S}{20}\right) = 40 + 90S\ (\text{ms})$$

同分支方案在输入阶段多花 `40 ms`，但在交换阶段每传递 `1 GB` 就能追回 `50 ms`。令两者相等，即可解出放置策略的精确分界点：

$$80 + 40S^* = 40 + 90S^* \implies 50S^* = 40 \implies S^* = 0.8\ \text{GB}$$

| 单轮卡间交换量 $S$ | 同分支放置（`GPU 0 + GPU 1`） | 跨分支放置（`GPU 0 + GPU 2`） | 更优策略 |
| --- | ---: | ---: | --- |
| `1.0 GB`（大于分界） | `120 ms` | `130 ms` | 同分支更快（P2P 省时胜出） |
| `0.1 GB`（小于分界） | `84 ms` | `49 ms` | 跨分支更快（独立上行胜出） |

如果每卡本地输入翻倍到 `2 GB` 而其余条件不变，输入阶段差距扩大为 `80 ms`，打平分界也同步右移到 `1.6 GB`。若框架支持“边拉输入边交换”的流水重叠，或消息小到微秒级，则需把阶段重叠与固定启动延迟重新排进时间线。

> [!tip] 工程直觉：按负载画像选放卡策略
> - **输入重、几乎不交换**（例如独立多副本推理、批量离线打分）：核心诉求是把 H2D / 权重加载做快，优先选**跨分支独立上行**，避免两张卡抢同一根 PCIe 管子。
> - **交换重、延迟敏感**（例如大模型张量并行推理 TP、强同步分布式训练）：卡间通信反复占据关键路径，必须优先收拢到**同 NVLink 域或同 Switch P2P 分支**。

## 用 `simpleP2P` 与 `nvbandwidth v0.11.0` 做功能与带宽取证

模型算清之后，交付现场通常用两类标准工具交叉取证：

- **功能连通性（`simpleP2P`）**：NCCL 2.27.3 排查文档推荐先跑 CUDA 示例 `simpleP2P`（或 `p2pBandwidthLatencyTest`）。它让两张卡分别运行访问对端显存的 Kernel 并把结果拷回主机比对，输出 `Test passed` 即证实本次覆盖的访问方向与数据校验均已跑通。
- **全对矩阵带宽（`nvbandwidth v0.11.0`）**：NVIDIA 官方开源基准工具，支持分别通过硬件**复制引擎（CE）**或 SM **Kernel** 搬运数据，`-b` 指定缓冲（**MiB**），带宽输出为十进制 **`GB/s`**。
  - `v0.11.0` 会在测前打印每张卡的 PCI BDF 与 Fabric Clique 健康状态（异常即告警）；默认开启数据校验（`--skipVerification` 可跳过），SM 用例按实际对齐字节统计带宽。

```bash
# 1. 记录工具版本与支持的测试用例清单（留存实测记录必备）
$ ./nvbandwidth --version
nvbandwidth Version: v0.11.0

$ ./nvbandwidth -l | grep "device_to_device_memcpy"
device_to_device_memcpy_read_ce
device_to_device_memcpy_write_ce
device_to_device_memcpy_read_sm
device_to_device_memcpy_write_sm

# 2. 使用 512 MiB 缓冲并开启 -v 逐项详细输出，对比 CE 写与 SM 读单向带宽
$ ./nvbandwidth -b 512 -v -t device_to_device_memcpy_write_ce device_to_device_memcpy_read_sm
Device 0: NVIDIA H100 PCIe (0000:41:00.0)
Device 1: NVIDIA H100 PCIe (0000:42:00.0)
...
Running device_to_device_memcpy_write_ce.
memcpy CE GPU(row) <- GPU(column) bandwidth (GB/s)   # 注意：结合 -v 逐项源/目的核实真实载荷方向
           0         1         2         3
 0      0.00     25.12       N/A       N/A           # 0 与 1 同 Switch P2P 畅通；跨分支无 P2P 被跳过
 1     25.08      0.00       N/A       N/A
 2       N/A       N/A      0.00     25.15
 3       N/A       N/A     25.11      0.00
```

> [!warning] 常见误区：读懂 `nvbandwidth` 矩阵的两个关键陷阱
> 1. **切勿盲信表头箭头方向**：`device_to_device_memcpy_write_ce` 与 `device_to_device_memcpy_read_sm` 均以发起侧为矩阵行、对端为列（`write_ce` 把行设备写入列设备，`read_sm` 从列设备读向行设备）。历史上（包括 `v0.10.0`）表头打印的箭头曾与有效数据流向相反，核对有向带宽时务必跟踪源/目的缓冲并用 `-v` 的逐项日志交叉印证。
> 2. **单对矩阵不代表并发总吞吐**：上述矩阵跳过对角线与未启用 P2P 的设备对，且每个有效格子是**逐对串行单测**出来的；要验证共享上行或整机并发能力，必须运行对应的并发用例或用共同主机时钟测共同完成时间，并在归档时附齐拓扑、版本、命令、消息大小与原始输出。

## 章末综合实战检验（无需实机）

下面这道综合题串联了本章全部四项核心能力。请先阅读已知条件与设备映射表，独立推演后再核对参考推理。

### 已知条件与场景映射

沿用双路四卡参考节点（`Switch A` 下挂 `GPU 0 / GPU 1 / NIC 0` 接 `Socket 0 / M0`；`Switch B` 下挂 `GPU 2 / GPU 3 / NIC 1` 接 `Socket 1 / M1`），跨分支试验的现场采集条件如下：

| 观察维度 | 已知状态与参数 |
| --- | --- |
| 进程卡号映射 | `cuda:0` → `0000:c1:00.0`；`cuda:1` → `0000:41:00.0` |
| 输入配置与页落点 | 两路输入各 `1 GB`；两线程 CPU 亲和均已绑在各卡本地，但 `numa_maps` 显示两路 DMA 源缓冲物理页**全部位于 `M0`** |
| 链路与有效预算 | 上行 A、B 均为健康 Gen4 x16；并行运行时本地 H2D/D2H 可达 `25 GB/s`，跨 Socket 远端 H2D 可达 `20 GB/s`，其余资源充足 |
| 卡间交换与对照方案 | 计算后需将 `GPU 0` 的 `0.7 GB` 结果交付给另一张卡；对照用的同分支方案（`GPU 0 + GPU 1`）两路输入仍均在本地 `M0` |

### 检验问题

1. **身份与路径还原**：还原 `cuda:0` 对应的物理 GPU、其 H2D 有效载荷完整路径以及就近网卡候选，并说明如何用桥树与映射页记录验证这条路径。
2. **放置比较与条件反转**：分别画出同分支直达与跨分支中转的卡间路径，计算并比较两种放置的本轮耗时；如果把 `cuda:0` 的输入页修正到本地内存，选择会如何变化？如果持续负载下测出上行 B 实际降宽为 Gen4 x8，原题设的 `20 GB/s` 是否还能成立？
3. **分块双槽流水推导**：保留 `cuda:0` 的远端输入条件不变，将跨分支的 `0.7 GB` 交换改为 **7 个 `0.1 GB` 的块**，用两个锁页缓冲槽轮换且两段可重叠并维持各自速率，计算交换最少需要多久以及本轮总耗时。
4. **“真走直达”与 HGX Fabric 判据**：同分支测试报告对端显存读取与校验通过，要进一步确认载荷全程留在 `Switch A` 内部还缺什么证据？再对照另一台 HGX H100 节点：若目标对中一卡 Fabric 为 `Completed / Success`、另一卡为 `In Progress / N/A`，且 FM 服务为 `active`，还需哪些证据才能确认两卡的 NVLink 通信可用？

### 分步参考推理

1. **身份、路径与验证依据**
   - **物理定位**：查映射表，`0000:c1:00.0` 对应宿主机 `GPU 2`，沿 `lspci -D -t` 桥树回溯至 `Switch B` 与 `Socket 1（NUMA 1）`。
   - **H2D 载荷路径**：尽管提交线程绑在 `Socket 1`，但参与 DMA 的物理页位于 `M0`，因此有效载荷走 **`M0 → Socket 0 → 处理器间互联 → Socket 1 根侧 → 上行 B → Switch B → GPU 2`**。
   - **验证方法与就近网卡**：按虚拟地址对照 `/proc/<pid>/maps`、`numa_maps` 与锁页缓冲区范围，确认采样时刻与 DMA 使用的是同一批页；沿桥树查得 `NIC 1`（`0000:c3:00.0`）与 `GPU 2` 同在 `Switch B` 下（`PIX`），是最优就近网卡候选。

2. **放置总耗时对比、条件反转与降宽复核**
   - **同分支方案（`GPU 0 + GPU 1`）**：两路各 `1 GB` 本地输入同向共用上行 A，耗时 $2 / 25 = 80\ \text{ms}$；交换走 `Switch A` 内部 P2P（`GPU 0 → Switch A → GPU 1`），耗时 $0.7 / 25 = 28\ \text{ms}$；本轮合计 **`108 ms`**。
   - **跨分支方案（`cuda:0` 输入仍在 `M0`）**：两路输入分走上行 A（`40 ms`）与上行 B（跨 Socket 远端 H2D `50 ms`），输入阶段耗时 $\max(40,\, 50) = 50\ \text{ms}$。
     - 交换经 `M0` 串行中转（D2H $0.7 / 25 = 28\ \text{ms}$，跨路 H2D $0.7 / 20 = 35\ \text{ms}$，合计 `63 ms`），本轮共 **`113 ms`**，此时同分支胜出。
   - **输入页迁回本地 `M1` 后的反转**：`GPU 2` 变为本地 H2D（`40 ms`），跨分支输入阶段共同耗时降为 `40 ms`，加上 `63 ms` 交换合计 **`103 ms` < `108 ms`**，跨分支反超胜出。
   - **上行 B 降宽为 Gen4 x8**：按第 02 篇公式，Gen4 x8 编码后单向上界仅约 **`15.75 GB/s`**，低于题设的 `20 GB/s`，因此原 `20 GB/s` 假设失效，所有经过上行 B 的预算必须推倒重算。

3. **7 块双槽流水的最小交换时间**
   - 每块 `0.1 GB` 的 D2H 耗时 $t_1 = 0.1 / 25 = 4\ \text{ms}$，跨 Socket H2D 耗时 $t_2 = 0.1 / 20 = 5\ \text{ms}$。
   - 代入两段流水公式 $T_{\text{pipe}}(7) = \min(4,\, 5) + 7 \times \max(4,\, 5) = 4 + 35 = \mathbf{39\ \text{ms}}$（第 1 块先用 `4 ms` 写入槽 0，随后较慢的 H2D 连续运行 `35 ms`，D2H 始终提前备好下一块并在槽释放后复用）。
   - 加上原远端输入阶段 `50 ms`，跨分支本轮总耗时降至 **`89 ms`**，反而比同分支 `108 ms` 更快。

4. **路由归因与 HGX H100 Fabric 完备证据**
   - **确认载荷留在 `Switch A` 内部**：对端 Kernel 读取并校验通过仅证明 P2P 功能可用；要归因到 Switch 内部转发，还须核对沿途 PCIe 桥的 `ACSCtl`（`ReqRedir-`、`CmpltRedir-`），并在平台支持计数器时对比两卡下行与 Switch 上行的实际流量。
   - **确认 HGX H100 两卡 NVLink 可用**：当前一卡已注册、另一卡仍在 `In Progress / N/A`，FM 服务 `active` 只说明进程在运行。
     - 须待另一卡 `Fabric State / Status` 也变为 `Completed / Success`，再用 `nvidia-smi topo -p2p n` 核对定向能力并实测校验；若程序含回退逻辑，还需链路计数确认载荷确实走了 NVLink。

## 全章知识串联与后续衔接

> [!summary] 第 04 章核心主线回顾
> 从主机内存页落点与 PCIe 桥树出发，先画出**数据候选路径**，再用权限查询、ACS/IOMMU 状态与 Fabric 注册日志确定**哪条路径真正畅通**，最后按链路方向与阶段依赖**给每轮字节记账**——这套三步法不仅能解释单机多卡的带宽波动与放卡取舍，也是后续理解集群通信的基石。

掌握单机多卡的数据路径后，我们正式迈入第二阶段“多机协同与数据供给”：
- [第 05 章 为什么一个作业需要多张 GPU](../05_为什么一个作业需要多张GPU/00_内容大纲.md)：从单机搬运转向作业视角，拆解训练并行（DP/FSDP/TP/PP/EP）与推理架构（TP/EP/PD 分离）为什么必须交换这些数据；
- [第 06 章 从普通网络通信走向 RDMA](../06_从普通网络通信走向RDMA/00_内容大纲.md)：把本章的 PCIe 路径延伸到高速网卡，讲透 GPUDirect RDMA 如何让网卡绕开主机内存直达显存；
- [第 08 章 NCCL 如何组织 GPU 间的通信](../08_NCCL如何组织GPU间的通信/00_内容大纲.md)：看清集合通信库如何自动探测本章学过的 NVLink、PCIe P2P、SHM 与跨机网络并动态选路。

> [!info]- 参考资料
> - **CUDA 异步执行与多设备同步边界**：[CUDA 13.2.1 · Asynchronous Execution](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/02-basics/asynchronous-execution.html)、[Multi-Device Stream, Event, and Memory Copy Behavior](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/03-advanced/multi-gpu-systems.html#multi-device-stream-event-and-memory-copy-behavior)
> - **NCCL P2P 功能验证参考**：[NCCL 2.27.3 · Troubleshooting (`simpleP2P` & GPU-to-GPU)](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html)
> - **`nvbandwidth v0.11.0` 官方文档与源码实现对照**：[NVIDIA nvbandwidth v0.11.0 · README](https://github.com/NVIDIA/nvbandwidth/blob/v0.11.0/README.md)、[v0.11.0 Release Notes](https://github.com/NVIDIA/nvbandwidth/releases/tag/v0.11.0)；历史版本计时校验与矩阵方向源码对照见 [v0.10.0 `memcpy.cpp#L512-L550`](https://github.com/NVIDIA/nvbandwidth/blob/v0.10.0/memcpy.cpp#L512-L550)、[`testcases.cpp#L282-L345`](https://github.com/NVIDIA/nvbandwidth/blob/v0.10.0/testcases.cpp#L282-L345)、[`testcase.h#L75-L110`](https://github.com/NVIDIA/nvbandwidth/blob/v0.10.0/testcase.h#L75-L110)
