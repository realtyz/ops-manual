# nccl-tests 与 perftest 的基准解读

四卡 AllReduce 跑通之后，排障与验收立刻面临两个维度的问题：整组跑完一次集合操作究竟多快？而跨节点那条 RDMA 物理路径单独跑起来又能交付多少吞吐？

打开压测报告，初学者往往会被三个带宽数字绕晕：集合测试同时给出 `algbw = 13.42 GB/s` 和 `busbw = 20.13 GB/s`，而底层单链路测试却跑出 `180 Gb/s`。本篇把 `nccl-tests`（集合级基准）与 `perftest`（双端点 Verbs 基准）的统计口径彻底对齐，教你用这两把尺子层层锁定慢因。

## 运行 `nccl-tests`：先核对成员与口径

下面用 `Open MPI 5.0.6` 在 `gpu-a` 与 `gpu-b` 上启动 `nccl-tests v2.17.9` 的 `all_reduce_perf`（多节点编译需开启 `make MPI=1`，运行时加载 `NCCL 2.27.3` 与 `CUDA 13.2.1`），扫描从 `8 B` 到 `64 MiB` 的全尺寸曲线：

```bash
# 锁定设备枚举序、引导接口与 RoCEv2 动态 GID 过滤条件
export CUDA_DEVICE_ORDER=PCI_BUS_ID
export NCCL_SOCKET_IFNAME='=eno1'
export NCCL_IB_HCA='=mlx5_0:1'
export NCCL_IB_ADDR_FAMILY=AF_INET
export NCCL_IB_ADDR_RANGE=198.51.100.0/24
export NCCL_IB_ROCE_VERSION_NUM=2

# 启动 4 个 MPI 进程（每节点 2 进程、每进程 1 线程 1 GPU），并将环境变量同步透传
mpirun -np 4 --host gpu-a:2,gpu-b:2 --map-by ppr:2:node --rank-by fill \
  -x CUDA_DEVICE_ORDER -x NCCL_SOCKET_IFNAME -x NCCL_IB_HCA \
  -x NCCL_IB_ADDR_FAMILY -x NCCL_IB_ADDR_RANGE -x NCCL_IB_ROCE_VERSION_NUM \
  ./build/all_reduce_perf \
    -t 1 -g 1 \          # 每进程 1 线程(-t)、每线程 1 卡(-g)，未拆分时总 Rank = 4×1×1 = 4
    -b 8 -e 64M -f 2 \   # 消息从 8 B 按 2 倍步进扫描到 64 MiB
    -d float -o sum \    # 数据类型 float32(4 B)，归约算子 sum
    -w 5 -n 20 \         # 5 次预热(-w)，20 次正式计时迭代(-n)
    -a 1 -c 1            # 多进程计时取均值(-a 1)，启用独立结果校验(-c 1)
```

看性能数字之前，先看清 `all_reduce_perf` 输出的**设备映射头**与**结果表列结构**（下方摘取四个代表消息尺寸）：

```text
# Using devices
#  Rank  0 Group  0 Pid 4100 on gpu-a device  0 [0x41] NVIDIA GPU  # 先逐行确认主机、CUDA 序号与 PCI 总线号
#  Rank  1 Group  0 Pid 4101 on gpu-a device  1 [0x61] NVIDIA GPU
#  Rank  2 Group  0 Pid 4200 on gpu-b device  0 [0x41] NVIDIA GPU
#  Rank  3 Group  0 Pid 4201 on gpu-b device  1 [0x61] NVIDIA GPU
#
#                                                              out-of-place                       in-place
#       size         count      type   redop    root     time   algbw   busbw #wrong     time   algbw   busbw #wrong
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)            (us)  (GB/s)  (GB/s)
           8             2     float     sum      -1    18.50    0.00    0.00      0    18.20    0.00    0.00      0
        4096          1024     float     sum      -1    22.40    0.18    0.27      0    22.10    0.19    0.28      0
     4194304       1048576     float     sum      -1   340.00   12.34   18.50      0   338.00   12.41   18.61      0
    67108864      16777216     float     sum      -1   5000.0   13.42   20.13      0   4980.0   13.48   20.21      0
```

表格左右拆为两组列：**`out-of-place`**（独立收发缓冲）与 **`in-place`**（收发共用同一显存缓冲原地归约）。横向对比不同轮次数据时，必须锁死同一组列与相同计时参数，并首先确认校验列 **`#wrong = 0`**——结果都不对的速度毫无意义。

> [!tip] 工程直觉：训推验收各盯哪一段曲线？
> - **在线推理 TP 验收**：紧盯 `4 KiB – 4 MiB` 行的 **`time (us)` 延迟列**，这一区间受固定同步开销主导，直接决定 Decode 阶段的 TPOT 延迟表现。
> - **分布式训练验收**：紧盯 `64 MiB` 及以上大消息坪区的 **`busbw (GB/s)` 列**，观察其是否贴近硬件有效带宽上界。

## 核心口径：`algbw` 与 `busbw` 如何复算

集合通信基准之所以同时列出两个带宽指标，是为了分别回答“应用视角搬得有多快”和“硬件互联被榨干了多少”这两个问题：

- **`algbw`（Algorithm Bandwidth，算法带宽）**：用逻辑数据大小 $S$ 直接除以单次集合耗时 $t$。它体现上层应用感知到的交付速度，但不考虑多卡之间实际要经过几轮转发。
- **`busbw`（Bus Bandwidth，等效总线带宽）**：将 `algbw` 乘以与操作类型和成员数 $P$ 相关的折算系数，还原出在最优环形算法下每张卡单方向承受的等效硬件载荷速率，专门用来和 NVLink、InfiniBand 或 RoCE 物理线速横向对标。

取上方输出中 `64 MiB`（$16{,}777{,}216$ 个 `float`，即 $67{,}108{,}864\text{ B}$）的 `out-of-place` 行，单次平均耗时 $t = 5000\,\mu\text{s}$，换算为十进制 `GB/s`：

$$t = 5000 \times 10^{-6}\text{ s} = 0.005\text{ s}$$

$$\text{algbw} = \frac{S}{t} = \frac{67{,}108{,}864\text{ B}}{0.005\text{ s} \times 10^9} \approx 13.42\text{ GB/s}$$

$$\text{busbw} = \text{algbw} \times \frac{2(P - 1)}{P} = 13.42 \times \frac{2 \times (4 - 1)}{4} \approx 20.13\text{ GB/s}$$

不同集合原语的逻辑大小 $S$ 含义和折算系数各不相同，对照 `nccl-tests v2.17.9` 源码约定汇总如下：

| 集合操作 | 公式中的基准大小 `size`（$S$）定义 | `busbw / algbw` 折算系数 |
| --- | --- | ---: |
| **AllReduce** | 每 Rank 的完整输入大小（等于完整输出大小） | $\dfrac{2(P - 1)}{P}$ |
| **AllGather** | 每 Rank 的完整**输出**大小（等于单卡本地输入的 $P$ 倍） | $\dfrac{P - 1}{P}$ |
| **ReduceScatter** | 每 Rank 的完整**输入**大小（等于单卡本地结果的 $P$ 倍） | $\dfrac{P - 1}{P}$ |

> [!warning] 常见误区：别把 `busbw` 直接等同于网卡实测流量
> 1. **同名 `size` 含义有别**：同样一行写着 `64 MiB`，AllGather 表示每张卡最终拼出的完整输出是 `64 MiB`（单卡输入只有 `16 MiB`），而 AllReduce 表示每张卡的输入就是 `64 MiB`。
> 2. **公式系数固定不变**：`nccl-tests` 对 AllReduce 一律按 Ring 系数 $2(P-1)/P$ 折算 `busbw`——即使实际跑了 Tree 算法，或出现了上一篇中两条跨机边挤爆单网口的交错环，公式本身也不会感知。要确认某个网口的真实压力，必须配合通道日志和硬件端口计数器。

## 逐级拆解：用 `perftest` 与不同规模对照定位慢因

假如四卡 AllReduce 的 `busbw` 远低于 `25 GB/s` 理论线速，不要直接一头扎进调优参数，而应**从最干净的双端点单向传输起步，逐层叠加复杂度**。

先抽离出跨机边 `1 → 2` 对应的那一对 RoCEv2 端口，用 `perftest 26.07.8` 的 `ib_write_bw` 跑一轮最纯粹的单向 RC Write 基线（`-x 3` 为两节点已核实的 RoCEv2 GID 索引，`192.0.2.12` 仅用于初期 TCP 建连参数交换）：

```bash
# 【第 1 轮：主机内存基线】验证底层 Verbs 路径与物理网络是否达标
# 在 gpu-b（服务端）启动：绑定 mlx5_0 端口 1、GID 3、RC 连接、4096 B MTU、8 MiB 消息、测 10 秒
ib_write_bw -d mlx5_0 -i 1 -x 3 -c RC -m 4096 -s 8388608 -q 1 -D 10 --report_gbits

# 在 gpu-a（客户端）启动：向 gpu-b 单向持续写入，输出单位为 Gb/s
ib_write_bw -d mlx5_0 -i 1 -x 3 -c RC -m 4096 -s 8388608 -q 1 -D 10 --report_gbits 192.0.2.12

# 【第 2 轮：GPU 显存直达基线】固定端口、方向与消息，追加显存与 DMA-BUF 参数直测 GDR
# gpu-b 服务端（Rank 2 使用本机 GPU 0）追加： --use_cuda=0 --use_cuda_dmabuf
# gpu-a 客户端（Rank 1 使用本机 GPU 1）在目标 IP 前追加： --use_cuda=1 --use_cuda_dmabuf
```

如果第 1 轮主机内存跑满接近 `200 Gb/s`，第 2 轮 GPU 显存却在注册阶段直接失败或速度暴跌，问题瞬间收窄到特定 GPU 与网卡的 PCIe 路径或 DMA-BUF / `nvidia-peermem` 显存注册栈（反向及另一组 GPU 配对也需分别验证）。把双端点与多规模集合测试排成五级阶梯，每一层的验证范围一目了然：

| 递进测试层级 | 本轮能够单独隔离和验证的范围 |
| --- | --- |
| 1. 双端点 `ib_write_bw`（主机内存） | 指定方向、网口、GID、MTU 与 QP 下的基础 RDMA 网络和物理链路健康度 |
| 2. 双端点 `ib_write_bw`（GPU 显存） | 叠加验证指定 GPU 与网卡配对的显存注册后端与单向 GDR 吞吐极限 |
| 3. 单机内部双卡 `all_reduce_perf` | 隔离单节点内两张 GPU 的集合计算与 P2P/SHM 机内传输，不含外部网络 |
| 4. 两机各一卡 `all_reduce_perf` | 隔离跨机双成员下的 NCCL 网络后端选择、引导建连与双端 GDR 表现 |
| 5. 两机各两卡 `all_reduce_perf` | 全量覆盖机内边、跨机边、算法通道切分与共享单网口的并发压力 |

遇到“两卡正常、四卡突然掉速”时要特别留意：成员数从 2 扩到 4，NCCL 可能会顺势切换算法、协议或通道分配。此时应在**固定四卡规模**上做单变量对照——例如保持消息、节点和网口不变，仅调整 Rank 到 GPU 的绑定顺序，同时核查通道日志确认跨机边确实换到了另一张物理卡。

如需单独检验 GDR 显存直达的收益，可在纯 PCIe 平台上固定 `NCCL_NET=IB`，分别跑默认 GDR 与 `NCCL_NET_GDR_LEVEL=LOC`（禁用 PCIe 跨设备显存直达）。具体操作上，先开详细日志确认 `/GDRDMA` 标记、算法与协议均符合对照设计，再关闭重日志单独采集一轮干净计时。

> [!note] 平台差异提示
> 在采用芯片间高速互联（如 Grace Hopper / Grace Blackwell 的 C2C 互联）的平台上，NCCL 判定 GDR 路径距离还会受对应 C2C 距离规则影响，设计 `NCCL_NET_GDR_LEVEL` 对照前应先核清节点互联形态。

## 用同时间窗的硬件端口计数交叉验真

当 `all_reduce_perf` 的 `busbw` 异常偏低、且通道日志怀疑出现多条跨机边挤爆同一个网口（例如上一篇的交错环）时，最有力的铁证来自**同一大消息稳定运行窗口内的硬件端口字节计数**。

在 Linux 网卡驱动下，有两类常用计数器可直接读取，务必注意它们的计量单位差异：

- **InfiniBand/RDMA 端口计数**：`/sys/class/infiniband/mlx5_0/ports/1/counters/` 下的 `port_xmit_data` 与 `port_rcv_data`，按 Linux 内核稳定 ABI 定义，**每 1 个计数单位代表 4 字节（32-bit word）**。若 2 秒稳态窗口内 `gpu-a` 发送计数净增 `11,200,000,000`，对应发送速率为：

$$R_{\text{tx}} = \frac{11{,}200{,}000{,}000 \times 4\text{ B}}{2\text{ s}} = 22.4 \times 10^9\text{ B/s} = 22.4\text{ GB/s}$$

- **`mlx5` 以太网扩展计数**：通过 `ethtool -S enp129s0f0` 读取的 `tx_vport_rdma_unicast_bytes` 与 `rx_vport_rdma_unicast_bytes`，直接以**字节（Byte）** 计量，无需乘 4。

采样前后先确认无显著背景流量，再把发送端的 TX 增量与对端接收口的 RX 增量交叉对齐。若发现四卡集合测试的 `busbw` 跌到约 `11 GB/s`，但同一窗口里物理网口的实际发送速率已经顶到 `22.4 GB/s` 贴近单口极限，再结合通道日志里同方向挂了两条跨机边——共享网口瓶颈的证据链就彻底闭环了。

现在，从职责分工、通信组生命周期、Ring/Tree 算法、物理选路到日志和基准核验，第 08 章的六块拼图已经齐备。下一篇我们通过五道贴近实战的综合检验题，把整章知识真正串活。

> [!info]- 参考资料
> - **nccl-tests 性能口径与源码**：[nccl-tests v2.17.9 · `PERFORMANCE.md`（`algbw`/`busbw` 公式）](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/doc/PERFORMANCE.md)、[nccl-tests AllReduce 节](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/doc/PERFORMANCE.md#allreduce)、[`README.md` 编译与参数](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/README.md)、[`src/common.cu` 设备与计时](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/common.cu)、[`src/all_reduce.cu`](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/all_reduce.cu)、[`src/all_gather.cu`](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/all_gather.cu)、[`src/reduce_scatter.cu`](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/reduce_scatter.cu)
> - **Open MPI 与 perftest 基准**：[Open MPI 5.0.6 `mpirun(1)`](https://docs.open-mpi.org/en/v5.0.6/man-openmpi/man1/mpirun.1.html)、[perftest `README`](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)、[perftest · `perftest_resources.c` 源码](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_resources.c)、[QP GID 与显存注册位置](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_resources.c#L2282)
> - **NCCL GDR 距离选项与排障**：[NCCL 2.27.3 `NCCL_NET_GDR_LEVEL`](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-net-gdr-level-formerly-nccl-ib-gdr-level)、[RoCE 排障指南](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html#rdma-over-converged-ethernet-roce)、[`src/graph/paths.cc`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/graph/paths.cc)
> - **Linux 端口计数 ABI 与 mlx5 计数器**：[Linux Kernel 6.18 Stable ABI (`port_xmit_data`)](https://docs.kernel.org/6.18/admin-guide/abi-stable.html)、[Linux `mlx5` RDMA Counters](https://docs.kernel.org/networking/device_drivers/ethernet/mellanox/mlx5/counters.html)
