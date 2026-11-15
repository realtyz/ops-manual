
四卡 AllReduce 做完后，可以问两个不同的问题。整组做完一次操作花多久？跨节点的某条 RDMA 路径单独跑时能交付多少数据？`nccl-tests` 用集合操作回答前者，perftest 可以为指定端点、方向和缓冲区建立后者的基线。要将两类结果联系起来，先把参与者、每次操作的大小与实际路径对齐。 [nccl-tests v2.17.9 性能口径](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/doc/PERFORMANCE.md) · [NCCL 2.27.3 RoCE 排障](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html#rdma-over-converged-ethernet-roce)

## 先确认集合测试测了谁、测了多大

以下教学命令使用 `nccl-tests v2.17.9` 的 `all_reduce_perf`。多进程和多节点运行需用 `make MPI=1` 构建，按实际安装位置提供 `MPI_HOME`、`CUDA_HOME` 与 `NCCL_HOME`。示例假定 Open MPI 5.0.6 能访问 `gpu-a` 和 `gpu-b` 并拉起进程，两端均可从同一工作目录访问相同版本的可执行文件，运行时加载 NCCL 2.27.3。每个进程都看见本机两张 GPU，且 `NCCL_TESTS_SPLIT`、`NCCL_TESTS_DEVICE` 与 `NCCL_IB_GID_INDEX` 均未设置。接口名、二进制路径和软件组合需在目标环境核对。 [nccl-tests v2.17.9 编译与参数](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/README.md) · [设备选择与测试分组源码](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/common.cu) · [Open MPI 5.0.6 启动参数](https://docs.open-mpi.org/en/v5.0.6/man-openmpi/man1/mpirun.1.html)

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

未拆分的测试组中，总 Rank 数为进程数乘以每进程线程数（`-t`），再乘以每线程的 GPU 数（`-g`）。这里是 `4 × 1 × 1 = 4`。先查测试输出的设备表，确认每个 Rank 的主机、CUDA 序号与 PCI BDF 对得上。随后看消息范围。命令从 8 B 到 64 MiB 按倍数扫描，使用 `float` 求和，配置 5 次预热、20 次计时迭代和一次单独的结果校验。`-a 1` 在 MPI 计时归并时取平均，本例每进程恰为一 Rank。默认口径包含等待通信完成，再按迭代数求单次时间，以微秒输出。 [nccl-tests v2.17.9 参数](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/README.md) · [计时、归并与校验源码](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/common.cu)

对于 AllReduce，`size` 是每个 Rank 的输入大小，等于 `count × 元素字节数`。输出分 out-of-place（独立收发缓冲）和 in-place（同一缓冲原地归约）两组，每组都有 `time`、`algbw`、`busbw` 和 `#wrong`。比较时先选一致的列和计时方式，确认结果校验已启用且 `#wrong=0`。小消息的结果更容易显出固定开销，扫到大消息时可观察吞吐是否趋于稳定。 [AllReduce 尺寸及带宽实现](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/all_reduce.cu) · [测试行输出与校验](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/common.cu)

## 复算 `algbw` 和 `busbw`

取一行**教学算例**，使用四 Rank 的 out-of-place 列。每 Rank 输入 64 MiB，是 `16,777,216` 个 4 B 的 `float`。`redop=sum`，`time=5000 µs`，`#wrong=0`。将时间换成秒，带宽取十进制 GB/s。

```text
t     = 5000 × 10⁻⁶ s = 0.005 s
algbw = 67,108,864 B / 0.005 s / 10⁹ ≈ 13.42 GB/s
busbw = algbw × [2 × (4 − 1) / 4]   ≈ 20.13 GB/s
```

`algbw` 将一份输入大小除以完成时间。`busbw` 再用集合操作与 Rank 数对应的系数归一化。在均衡分块 Ring 中，四 Rank 的每个成员每次发送量为输入的 1.5 倍，因此这个系数能与该环的发送量对应。工具对 AllReduce 一律用此系数，选 Tree 时仍按同样规则计算。若要知道某个网口承担了多少负载，需要通道映射或同一时间窗的端口计数。在两种环顺序的算例中，交错环每个方向有两条跨机边，所以该方向的端口模型负载还要再乘 2。 [nccl-tests 带宽定义](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/doc/PERFORMANCE.md#allreduce) · [AllReduce 换算实现](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/all_reduce.cu)

| 操作 | 带宽公式中的数据大小 `S` | `busbw / algbw` |
| --- | --- | ---: |
| AllReduce | 每 Rank 的完整输入大小，等于完整输出大小 | `2(P−1)/P` |
| AllGather | 每 Rank 的完整输出大小，为本地贡献大小的 `P` 倍 | `(P−1)/P` |
| ReduceScatter | 每 Rank 的完整输入大小，为本地结果大小的 `P` 倍 | `(P−1)/P` |

所以两行同样写 64 MiB，AllGather 的每 Rank 输入与 AllReduce 的每 Rank 输入仍有不同含义。记录比较对象时附上操作、数据类型、Rank/GPU/NIC 分布、实际加载的库版本、计时和校验选项。 [AllGather 换算实现](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/all_gather.cu) · [ReduceScatter 换算实现](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/src/reduce_scatter.cu)

## 用双端点和不同规模的集合测试缩小范围

从主环的 `1 → 2` 取两个端点。用 perftest 26.07.8 在同一对 RoCEv2 端口上做单向 `ib_write_bw`，`gpu-a` 客户端写入 `gpu-b` 服务端。下例假设两端的 RoCEv2 GID 恰好都在本地索引 3，路径支持 4096 B QP MTU。结尾的 `192.0.2.12` 用于 TCP 参数交换，RDMA 载荷用 `-d`、`-i` 和 `-x` 所选端口及 GID。这是教学命令，实际 GID、MTU、地址与构建选项须按现场确认。 [perftest 固定版本的参数与运行方式](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README) · [perftest QP 源 GID 设置](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_resources.c)

```bash
# gpu-b，服务端，主机内存
ib_write_bw -d mlx5_0 -i 1 -x 3 -c RC -m 4096 \
  -s 8388608 -q 1 -D 10 --report_gbits

# gpu-a，客户端，主机内存，写入 gpu-b
ib_write_bw -d mlx5_0 -i 1 -x 3 -c RC -m 4096 \
  -s 8388608 -q 1 -D 10 --report_gbits 192.0.2.12
```

这轮让一个 RC（Reliable Connection，可靠连接）QP 反复传 8 MiB 的消息，按 10 秒时长模式测量，输出载荷速率用 Gb/s。第二轮固定端口、方向和消息，只将两端缓冲区换成 GPU 显存。对于支持 CUDA DMA-BUF 的构建，在客户端参数中的目标地址之前、服务端的选项中分别添加 `--use_cuda=<本机CUDA序号> --use_cuda_dmabuf`，将占位符替换成对应设备的可见序号。按本章默认映射，A 端选 GPU 1，B 端选 GPU 0。查选卡、显存注册和传输完成记录，再解释显存轮的结果。若主机轮成功、显存轮在注册时失败，优先查 GPU/NIC 配对和注册后端。反方向及其它 GPU/NIC 配对要另测。 [perftest CUDA 与 DMA-BUF 条件](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README) · [显存注册实现](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_resources.c#L2282)

| 测试 | 这轮能验证的范围 |
| --- | --- |
| 双端点 `ib_write_bw`，主机缓冲 | 选定的方向、端口、消息大小和 QP 数下的基本 Verbs 路径。 |
| 双端点 `ib_write_bw`，GPU 缓冲 | 同样条件下，增加对指定 GPU/NIC 配对、显存注册后端和载荷方向的覆盖。 |
| 每个节点各做一次本地双卡 AllReduce | 本机两 GPU 的集合结果与性能，可配合通道日志识别 P2P 或 SHM。 |
| 两节点各一卡 AllReduce | 两个选定成员的跨机集合基线，可配合日志确认 NCCL 后端和两端路径。 |
| 两节点各两卡 AllReduce | 四张卡共同执行时的结果与性能，涉及机内传输、跨机传输和共享资源。 |

若两 Rank 正常，四 Rank 才慢，增加成员也可能改变算法、协议、通道及其工作分配。先查四 Rank 的实际通道及新加入的 GPU，再在固定规模上做对照。例如保持四 Rank、消息、节点及端口一致，调整 Rank 与 GPU 的对应关系，并用日志核实跨机边是否换到了另一张 GPU。如果重排后还是同一物理路径，这轮就没有改变待检验的配对。

如需定位显存直达的影响，可在本例的纯 PCIe 平台上固定 `NCCL_NET=IB`，对照默认 GDR 与 `NCCL_NET_GDR_LEVEL=LOC`。用通道及 TUNING 日志核对收发的 `/GDRDMA` 标识、算法、协议和通道是否符合对照设计，单独采一轮轻日志计时。C2C 互联平台的距离规则还受对应 GDR 选项影响，使用同一对照前需核对平台条件。 [NCCL GDR 距离与 C2C 选项](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-net-gdr-level-formerly-nccl-ib-gdr-level) · [GDR 路径实现](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/graph/paths.cc)

## 用同一时间窗的端口计数解释共享

固定在大消息区间取样，记录两端端口的发送与接收增量。若驱动在 `/sys/class/infiniband/mlx5_0/ports/1/counters/` 实现了 `port_xmit_data` 和 `port_rcv_data`，Linux ABI 给这两个计数的口径是每计数单位 4 B。例如 2 秒内发送计数增加 `11,200,000,000`，则发送速率为 `11,200,000,000 × 4 / 2 = 22.4 GB/s`。若改读 `ethtool -S enp129s0f0` 的 mlx5 `tx_vport_rdma_unicast_bytes` 或 `rx_vport_rdma_unicast_bytes`，这些值已经按字节计。采样前后要确认设备、计数范围、背景流量和窗口，用 A 的发送与 B 的接收相互校对。 [Linux 6.18 端口计数 ABI](https://docs.kernel.org/6.18/admin-guide/abi-stable.html) · [mlx5 RDMA 字节计数](https://docs.kernel.org/networking/device_drivers/ethernet/mellanox/mlx5/counters.html)

在交错环的理想模型里，同方向两条边的字节都要过同一个端口。若实际通道日志证实这个映射，且同一计时窗口内的端口负载接近该端口可达的基线，就有了共享端口限制的证据。端口速率还会包含其计数范围内的其它流量。结合各 Rank 的进度和收发两端计数，才能判断负载集中发生在哪段路径。
