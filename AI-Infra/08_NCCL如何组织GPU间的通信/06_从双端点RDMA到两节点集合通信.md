# 从双端点 RDMA 到两节点集合通信

两台机器的 `ib_write_bw` 跑通后，四张 GPU 的 AllReduce 仍可能慢。双端点测试只经过选定的两个端口和缓冲区，集合还要协调所有 Rank、经过机内边，并让多条边共享设备资源。先隔离一条 RoCEv2 路径，再增加参与者，最后用固定规模的对照检验具体假设。 [NCCL 2.27.3 的 RoCE 排障步骤](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html#rdma-over-converged-ethernet-roce)

## 固定一条 RoCEv2 路径，换两种缓冲区

以 `gpu-a` 和 `gpu-b` 为例，两端的 `mlx5_0:1` 都对应 `enp129s0f0`，本地 RoCEv2 地址为 `198.51.100.11` 与 `198.51.100.12`。先按 `gids`、`gid_attrs/types` 和 `gid_attrs/ndevs` 核对实际表项。以下教学命令假设两端恰好都用索引 `3`，且整条以太网路径支持所选的 4096 B QP MTU。`ib_write_bw` 的客户端在 `gpu-a`，写入 `gpu-b`。命令末尾的 `192.0.2.12` 用于 TCP 建连和交换参数，Verbs 数据路径使用 `-d`、`-i` 和 `-x` 选定的端口与 GID。两端使用同一 perftest 版本。 [perftest 26.07.8 参数和运行方式](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README) · [perftest 的 QP 源 GID 选择](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_resources.c#L3550)

```bash
# gpu-b，服务端，主机内存
ib_write_bw -d mlx5_0 -i 1 -x 3 -c RC -m 4096 \
  -s 8388608 -q 1 -D 10 --report_gbits

# gpu-a，客户端，主机内存，写入 gpu-b
ib_write_bw -d mlx5_0 -i 1 -x 3 -c RC -m 4096 \
  -s 8388608 -q 1 -D 10 --report_gbits 192.0.2.12
```

第二轮保持同一端口、GID、方向、RC QP、消息和时长。用支持 CUDA DMA-BUF 的构建时，在两端各自命令的参数区加 `--use_cuda=0 --use_cuda_dmabuf`，先按实际 CUDA 序号核对 GPU BDF，并用 `--help` 验证参数可用。DMA-BUF 还需要 GPU 驱动、内核、RDMA provider 和 NIC 支持。确认实际选择了设备显存、MR 注册成功且传输完成，再把这一轮记为所选 GPU/NIC 对的 GDR 基线。反方向需要调换客户端和服务端另测，其他 GPU/NIC 配对也要分别确认。 [perftest GPU 与 DMA-BUF 使用条件](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README) · [DMA-BUF MR 注册实现](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_resources.c#L2282)

如果主机缓冲通过、显存缓冲失败，第一轮已经验证了该端口和方向上的基本 Verbs 路径。第二轮若在注册阶段报错，先看选卡和注册后端。若已完成注册却在传输时报错，再查方向、设备路径和 QP 完成错误。记录两端命令、退出状态和完整输出，避免把不同阶段的失败合成一个带宽数值。

## 增加 Rank，再做固定规模的对照

下表按证据范围安排测试。集合组统一用 `all_reduce_perf`，每进程一张 GPU，固定相同的消息扫描、类型、迭代和缓冲区列。每种规模记录实际设备表与选路日志，另外做轻日志的计时轮次。 [nccl-tests 多进程参数与输出](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/README.md)

| 测试 | 端点与缓冲区 | 得到的证据 |
| --- | --- | --- |
| `ib_write_bw` 主机缓冲 | `gpu-a` → `gpu-b`，同一对 RoCEv2 端口，单向 RC Write | 指定方向、消息和 QP 数下的双端点 Verbs 载荷吞吐。 |
| `ib_write_bw` GPU 缓冲 | 同样的端口和方向，换两端缓冲区 | 对应 GPU/NIC、显存注册机制和方向下的 GDR 路径及吞吐。 |
| 每节点分别双卡 AllReduce | 每轮仅一个节点，两张 GPU | 对应两张 GPU 的机内集合正确性和性能，结合日志辨认 P2P 或 SHM。 |
| 两节点各一卡 AllReduce | 两 Rank，选定的两个 GPU | 第一个跨机集合基线，同时验证 NCCL 的网络后端与两端 GDR 选择。 |
| 两节点各两卡 AllReduce | 四 Rank，同一对 RoCEv2 端口 | 在机内与机间路径同时参与时，观察该规模的集合性能、通道映射和端口负载。 |

若前两轮 perftest 和跨机各一卡都正常，到了各两卡才变慢，先查四 Rank 新加入的 GPU 与通道边。两 Rank 到四 Rank 可能同时改变算法、协议、通道和消息分配，差值只表明扩大规模后出现差异。再做同为四 Rank 的对照，保留消息、CPU 放置、后端和 RoCE 端口，调整 GPU 的可见顺序或 Rank 绑定。先用日志确认跨机边实际换到了哪张 GPU，再看慢点是否随 GPU–NIC 配对移动。若 NCCL 重排后物理路径相同，这一轮没有改变目标变量。

若怀疑显存直达影响结果，可在本例的纯 PCIe 平台上，固定相同的 Rank 和消息，两轮都给各进程传入 `NCCL_NET=IB` 固定内置 Verbs 后端，再比较默认 GDR 选择和 `NCCL_NET_GDR_LEVEL=LOC`。用通道与 TUNING 日志核对 `NET/IB`、收发方向、算法、协议和通道。只有 `/GDRDMA` 确实按预期变化、其余关键路径仍可比较时，性能差才支持对该 GDR 路径的判断。2.27.3 的 C2C 平台还有额外的 GDR 条件，直接套用该对照前需核对其路径。 [GDR 拓扑与 C2C 选项](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-net-gdr-level-formerly-nccl-ib-gdr-level) · [GDR 检查实现](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/graph/paths.cc#L388)

## 用端口增量验证共享链路

200 Gb/s 全双工端口的单方向名义上限是 `200 ÷ 8 = 25 GB/s`。两条同向流共享该端口时，合计仍受这个上限约束，实际有效载荷还要扣除线上的协议开销。只在掌握实际 Rank 顺序和通道分配时，才能估算集合操作有多少载荷要过端口。

取一个**单通道 Ring 算例**。每 Rank 的输入都是 64 MiB，四 Rank 的每条环边在一次 AllReduce 中共发 `64 × 2 × 3/4 = 96 MiB`。如果按 `A0 → A1 → B0 → B1 → A0` 排列，每方向只有一条跨机边，光搬运这 96 MiB 就至少需要约 `4.03 ms`。如果通道顺序交错为 `A0 → B0 → A1 → B1 → A0`，每方向有两条跨机边，单端口该方向至少承担 192 MiB，时间下界变为约 `8.05 ms`。计算假设两端仍共享各自唯一的 200 Gb/s 端口，并忽略协议开销和其他等待。实际 NCCL 可用多个通道，先看每条通道上落到端口的工作，再做这种推算。 [AllReduce 传输量推导](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/doc/PERFORMANCE.md#allreduce)

采集时在同一个大消息测试窗口前后读两端的端口计数。若驱动在 `/sys/class/infiniband/mlx5_0/ports/1/counters/` 下提供并实现 `port_xmit_data` 和 `port_rcv_data`，按 ABI 将两个方向各自的计数差乘以 4，再除以经过的秒数。例如发送计数 2 秒增加 `11,200,000,000`，得到该窗口的发送速率 `22.4 GB/s`。若使用 `ethtool -S enp129s0f0` 中的 mlx5 `tx_vport_rdma_unicast_bytes` 或 `rx_vport_rdma_unicast_bytes`，它们已经按字节计数，直接除以秒数。先确认字段在该端口上可用、计数范围匹配，并记录背景流量。 [Linux 6.18 InfiniBand 端口计数 ABI](https://docs.kernel.org/6.18/admin-guide/abi-stable.html) · [mlx5 RDMA 计数定义](https://docs.kernel.org/networking/device_drivers/ethernet/mellanox/mlx5/counters.html)

同一窗口中，若通道映射显示两条同向边叠在同一口，且端口速率接近该路径的已知基线，共享端口就是一个有证据的容量约束。若速率明显偏低，则结合 GID 与传输日志、机内进度和 RoCE 队列或拥塞计数，看是哪一段先失去进度。`ib_write_bw` 的单向吞吐、`all_reduce_perf` 的归一化 `busbw` 和端口字节速率分别保留。它们只有在方向、路径和取样窗口对齐后，才能一起支撑瓶颈判断。
