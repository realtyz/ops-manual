# 沿 PCIe 路径算单向与并发带宽

两张 GPU 都以 Gen4 x16 接到同一个 PCIe Switch，仍可能经同一条 Gen4 x16 上行读取主机内存。判断并发能搬多少，要先沿每条路径找出实际经过的链路，再把经过同一段、同一方向的流量加起来。

## 先读链路状态，再换算速率

下面是双路四卡教学节点的 Switch A 分支节选。根端口 `0000:20:01.0` 经一条链路连接 Switch A 的上行端口 `0000:3f:00.0`。Switch 内部总线 `40` 下有三个下行端口，分别连到 GPU 0、GPU 1 和 NIC 0。分支 B 使用另一个根端口和独立上行。 [pciutils · lspci 手册](https://github.com/pciutils/pciutils/blob/master/lspci.man)

```text
# lspci -D -t 的教学节选，省略其他根总线
-[0000:20]---01.0-[3f-43]----00.0-[40-43]--+-01.0-[41]----00.0
                                           +-02.0-[42]----00.0
                                           \-03.0-[43]----00.0
```

读取 `lspci -D -s <BDF> -vv` 中的 `LnkCap` 和 `LnkSta`。前者给出该端口报告的最大能力，后者给出当前协商状态。按桥树查上游端口及各下行，链路两端对应同一物理段，只算一次。下表是人工整理的**窄上行变体**，假设在持续传输时采样。`nvidia-smi -q` 的 GPU Link `Current` 和 `Max` 可辅助检查 GPU 端链路，空闲时当前速率或宽度可能降低。Switch 上行仍要回到相应 PCIe 端口检查。 [pciutils · PCIe 字段解码源码](https://github.com/pciutils/pciutils/blob/master/ls-caps.c) · [NVIDIA SMI · GPU Link Information](https://docs.nvidia.com/deploy/nvidia-smi/)

| 采样的 PCI 功能 | 对应链路 | `LnkCap` 的 `Speed` / `Width` | `LnkSta` 的 `Speed` / `Width` |
| --- | --- | --- | --- |
| `0000:41:00.0` GPU 0 | GPU 0 ↔ Switch A 下行 | `16 GT/s` / `x16` | `16 GT/s` / `x16` |
| `0000:42:00.0` GPU 1 | GPU 1 ↔ Switch A 下行 | `16 GT/s` / `x16` | `16 GT/s` / `x16` |
| `0000:3f:00.0` Switch A 上行端口 | Switch A ↔ 根端口 | `16 GT/s` / `x16` | `16 GT/s` / `x8` |

Gen3、Gen4、Gen5 的每 Lane 速率分别为 `8`、`16`、`32 GT/s`，均使用 `128b/130b` 编码。对这三代，乘以 Lane 数和 `128/130`，再除以每字节 `8 bit`，就得到**编码后的单向速率上界**。这里 `1 GB = 10⁹ B`。事务包头、链路管理和主机侧资源还会进一步影响实际有效载荷吞吐。 [PCI-SIG · 速率与编码表](https://pcisig.com/sites/default/files/files/PCI-SIG-Seamless_Transition_to_PCIe_5.0_in_System_Implementations_FINAL.pdf)

| 协商状态 | 单向上界的计算 | 单向上界 |
| --- | --- | ---: |
| Gen3 x16 | `8 × 16 × 128/130 ÷ 8` | 15.75 GB/s |
| Gen4 x8 | `16 × 8 × 128/130 ÷ 8` | 15.75 GB/s |
| Gen4 x16 | `16 × 16 × 128/130 ÷ 8` | 31.51 GB/s |
| Gen5 x16 | `32 × 16 × 128/130 ÷ 8` | 63.02 GB/s |

在窄上行变体中，去往任一 GPU 的本地 H2D 都经过 x8 上行，单路有效吞吐也不可能达到 `20 GB/s`。两个方向同时传输时，Gen4 x16 的编码后速率可按双向合计写成 `63.02 GB/s`，其中任一方向仍以 `31.51 GB/s` 为上界。

## 共享链路按方向记账

以下回到健康的教学拓扑，GPU 下行与两条 Switch 上行均设为 Gen4 x16。**另设每段每方向的有效预算为 `25 GB/s`**，并假定内存系统和其他资源足以供给算例中的流量。对某一条流，沿路最小预算约束该流的速率。对多条流，把经过每段链路的负载按方向相加。共同完成时间至少为各段各方向 `总字节数 ÷ 对应有效预算` 中的最大值。

| 同时启动的两路输入 | 瓶颈链路承担的有效载荷 | 共同完成时间下界 |
| --- | --- | --- |
| M0 → GPU 0、M0 → GPU 1，各 `1 GB` | 上行 A 的主机→Switch 方向合计 `2 GB` | `2 ÷ 25 s = 80 ms` |
| M0 → GPU 0、M1 → GPU 2，各 `1 GB` | 上行 A、B 的主机→Switch 方向各 `1 GB` | `1 ÷ 25 s = 40 ms` |

第一种放置下，若题设再给定两路均分上行，两路各 `12.5 GB/s`。若没有分配策略，只能由共享约束推出总量和共同完成的下界。反过来，若 GPU 0 做 `18 GB/s` H2D，GPU 1 同时做 `20 GB/s` D2H，应把 `18` 和 `20` 记入上行 A 的两个方向。两个方向各未超过 `25`。是否还能同时达到目标，仍取决于共享的主机内存、复制引擎及协议开销等条件。

## 主机中转时，两段可能串行也可能重叠

GPU 0 把 `1 GB` 交给 GPU 1，若先完整写入 M0，再从 M0 完整读到 GPU 1，两段各 `25 GB/s`，忽略固定开销时是 `40 + 40 = 80 ms`，最终交付吞吐 `12.5 GB/s`。若切成 `10` 块，每块在任一段耗时 `4 ms`，并假定独立的可重叠操作、两块可轮换的主机缓冲区、正确的块间依赖以及足够的双向和内存预算，稳态可在一块 D2H 的同时对上一块 H2D。理想完成时间为 `(10 + 1) × 4 = 44 ms`，约 `22.73 GB/s`。 [CUDA 13.2.1 · Asynchronous Execution](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/02-basics/asynchronous-execution.html)

若设备对已确认走 Switch A 内的 PCIe P2P，按 `25 GB/s` 的同等单路预算交付 `1 GB` 只需 `40 ms`。流水改变两次搬运的时间安排，P2P 改变经过的链路。比较两种优化前，先从运行证据确认原有传输方式。
