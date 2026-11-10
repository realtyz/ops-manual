# 用 nccl-tests 测量集合操作

同一张结果表里，`time` 表示一次集合操作的时间，`algbw` 按每次操作的数据大小计算，`busbw` 再按集合类型和 Rank 数换算。读一行之前，先确认进程、GPU 和缓冲区是怎么组织的。以下用 `nccl-tests v2.17.9` 的 `all_reduce_perf`。 [nccl-tests v2.17.9 参数](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/README.md) · [带宽口径](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/doc/PERFORMANCE.md)

## 先看谁在测、测多大

`nccl-tests` 的多进程与多节点运行依赖 MPI。用 `make MPI=1` 构建，并按环境提供 `MPI_HOME`、`CUDA_HOME` 和 `NCCL_HOME`。单个未拆分的测试组内，总 Rank 数是进程数乘以每进程线程数（`-t`），再乘以每线程 GPU 数（`-g`）。本例两节点各两个进程，`-t 1 -g 1` 得到四个 Rank。用测试输出的设备表核对各 Rank 的主机与 GPU PCI BDF，尤其留意已有的 `CUDA_VISIBLE_DEVICES` 是否改变了可见设备。 [构建与 GPU 数量参数](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/README.md) · [Rank 与设备选择实现](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/common.cu)

下面是**教学命令**。假设 Open MPI 5.0.6 可以拉起 `gpu-a` 和 `gpu-b` 上的进程，两端在相同工作目录下有 `MPI=1` 构建的可执行文件，均加载 NCCL 2.27.3，两张 GPU 都对本地进程可见。接口与地址须按现场核对。CUDA 12.9 与 NCCL 2.27.3 的版本关系可查发行说明，这组完整的构建和运行组合仍需在目标节点验证。 [Open MPI 启动和放置](https://docs.open-mpi.org/en/v5.0.6/man-openmpi/man1/mpirun.1.html) · [NCCL 2.27.3 发行说明](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/release-notes/rel_2-27-3.html)

```bash
export CUDA_DEVICE_ORDER=PCI_BUS_ID
export NCCL_SOCKET_IFNAME='=eno1'
export NCCL_IB_HCA='=mlx5_0:1'
export NCCL_IB_ADDR_FAMILY=AF_INET
export NCCL_IB_ADDR_RANGE=198.51.100.0/24
export NCCL_IB_ROCE_VERSION_NUM=2

mpirun -np 4 --host gpu-a:2,gpu-b:2 --map-by ppr:2:node --rank-by fill \
  -x CUDA_DEVICE_ORDER -x NCCL_SOCKET_IFNAME -x NCCL_IB_HCA \
  -x NCCL_IB_ADDR_FAMILY -x NCCL_IB_ADDR_RANGE -x NCCL_IB_ROCE_VERSION_NUM \
  ./build/all_reduce_perf -t 1 -g 1 -b 8 -e 64M -f 2 \
  -d float -o sum -w 5 -n 20 -a 1 -c 1
```

`-b 8 -e 64M -f 2` 从 8 B 到 64 MiB 按倍数扫描。`-w 5` 设置预热轮数，`-n 20` 设置计时迭代数，`-a 1` 让 MPI 构建对各 Rank 的计时取平均，`-c 1` 另做一次正确性检查。这里采用默认的完成计时方式。计时包括等待已提交的操作完成，再换算为每次操作的时间，输出单位为微秒。小消息的时间更容易显出固定开销。扫到大消息后可以观察数据量与耗时是否接近线性，以及吞吐是否趋于稳定。 [nccl-tests 参数](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/README.md) · [v2.17.9 计时和校验源码](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/common.cu#L612)

## 用一行数据复算两个带宽

先读 `size`、`count`、`type` 和 `redop`。AllReduce 的 `size` 是每个 Rank 的输入大小，等于 `count × 每元素字节数`。输出分为 out-of-place 和 in-place 两组，前者使用分开的收发缓冲区，后者在同一个缓冲区原地归约。两组各有 `time`、`algbw`、`busbw` 和 `#wrong`。同一大小做对比时选同一组列，并确认已启用校验且 `#wrong=0`。 [nccl-tests AllReduce 的缓冲区与带宽实现](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/all_reduce.cu) · [计时和校验源码](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/common.cu#L631)

以下是**算例**。取四 Rank 的 out-of-place 列，假设 `size=67,108,864 B`，`count=16,777,216` 个 `float`，`redop=sum`，`time=5000 µs`，`#wrong=0`。将微秒换为秒，再用十进制的 GB/s 表示带宽。

```text
t     = 5000 × 10⁻⁶ s = 0.005 s
algbw = 67,108,864 B / 0.005 s / 10⁹ ≈ 13.42 GB/s
busbw = algbw × [2 × (4 − 1) / 4]   ≈ 20.13 GB/s
```

系数 `2(P−1)/P` 对应 AllReduce 在均衡分块 Ring 中每 Rank 的发送量与输入大小之比。两个阶段各传 `(P−1)/P` 份输入。工具把这个系数用作 AllReduce 的统一归一化，即使此次选了其他算法，`busbw` 仍按同一公式计算。要估计端口实际承担了多少流量，要看跨节点边如何映射到端口，再用同一时间窗口的端口计数验证。 [nccl-tests 带宽推导](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/doc/PERFORMANCE.md#allreduce) · [AllReduce 带宽计算](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/all_reduce.cu#L59)

| 操作 | 带宽公式中的数据大小 `S` | `busbw / algbw` |
| --- | --- | ---: |
| AllReduce | 每 Rank 的输入或输出数组大小，两者相同 | `2(P−1)/P` |
| AllGather | 每 Rank 最后收齐的完整输出大小，即 `P × 本地贡献大小` | `(P−1)/P` |
| ReduceScatter | 每 Rank 提交的完整输入大小，即 `P × 本地结果大小` | `(P−1)/P` |

同是 64 MiB，AllGather 的每 Rank 输入只是完整输出的一部分，ReduceScatter 的每 Rank 输出也只有完整输入的一部分。比较记录时，保存操作、数据类型、Rank 数、GPU/NIC 绑定、拓扑、软件版本、计时选项和输出列。把实际加载的 NCCL 版本与构建时所用版本分别记下。这样拿到两行数字时，可以先判断它们是不是同一口径，再讨论性能差异。 [AllGather 带宽实现](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/all_gather.cu#L38) · [ReduceScatter 带宽实现](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/reduce_scatter.cu#L37)
