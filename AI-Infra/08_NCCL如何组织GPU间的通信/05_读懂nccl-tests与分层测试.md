# 读懂 nccl-tests 与分层测试

## 测试先确定参与者与操作

`nccl-tests` 对给定集合操作检查性能和数据正确性。它绕过训练的数据加载、模型计算和优化器，因而适合验证通信组在某些条件下的行为。它仍会使用 GPU 执行、内存与通信资源，不能把结果看作仅测交换机。

先读以下**教学命令及其假定布局**，无需执行。基线为 nccl-tests v2.19.7 的普通内置集合实现，单线程、无 CUDA Graph、无 CPU 计时；二进制已正确链接目标 NCCL，设备访问及空闲显存足够。

```bash
# 单机一进程管理两张 GPU，只有一组两 Rank 通信器
./build/all_reduce_perf -b 1K -e 256M -f 2 -t 1 -g 2 -d float -o sum -w 5 -n 20 -c 1

# 多机：MPI 支持已编入；启动器保证每机两进程、每进程一张不同 GPU
mpirun -np 4 -N 2 -host nccl-a,nccl-b ./build/all_reduce_perf -b 1K -e 256M -f 2 -t 1 -g 1 -d float -o sum -w 5 -n 20 -c 1
```

第二条借用 Open MPI 风格，SSH/进程启动、环境传播、设备和 CPU 放置都是已给定前提，不是通用部署脚本。四个实际 Worker 的设备映射必须与第一篇身份表相同，不能只由命令里的 `-np 4` 推断。

测试 Rank 总数为 `进程数 × 每进程线程数 × 每线程 GPU 数`。所以第一条是 `1×1×2=2`，第二条是 `4×1×1=4`；若把第二条也写成 `-g 2`，就会创建八个测试 Rank，而非原来的四个，且每机的不同进程可能重复使用 GPU。测试程序负责建立并调用 NCCL；MPI 在此承担多进程组织和相关协调，不能因此认定集合负载由 MPI 搬运。

`-b/-e` 是扫描大小，`-f 2` 逐次翻倍，`-w 5` 为预热次数，`-n 20` 为计时迭代数，`-c 1` 启用相应轮次的数据检查。`1K`、`256M` 使用二进制单位；这不是 256 MB。完整参数参考 [nccl-tests v2.19.7 README：Usage / Arguments](https://github.com/NVIDIA/nccl-tests/blob/v2.19.7/README.md)。消息大小还不是整个进程显存占用：独立收发、期望结果、通信内部缓冲都会占用空间。

## 一行输出有两组独立结果

以下是**自拟、删节输出**，列名按该测试组织；只展示扫描中的三档。默认报告方式不额外改为最大/最小 Rank 计时。各档均已完成并启用校验，数字用于复算，不是 GPU 性能记录。

```text
#                         out-of-place                       in-place
# size       count     type  redop root   time  algbw busbw #wrong   time  algbw busbw #wrong
# (B)        elements                    (us) (GB/s)(GB/s)         (us) (GB/s)(GB/s)
1024         256       float sum   -1     40.0   0.03  0.04    0     39.0   0.03  0.04    0
1048576      262144    float sum   -1    200.0   5.24  7.86    0    190.0   5.52  8.28    0
268435456    67108864  float sum   -1  25000.0  10.74 16.11    0  24000.0  11.18 16.78    0
```

先检查 `count × 4 B = size`，例如 `67108864×4=268435456`。AllReduce 的 root 为 `-1`，表示该列对无根操作不适用，不是有一个 Rank −1。

**Out-of-place** 使用不同输入、输出缓冲；**in-place** 将结果写回原缓冲。后面的 time、带宽与错误列各自成组，不可以拿左侧时间配右侧带宽。In-place 减少显式独立收发缓冲的需求，并不承诺所有内部临时空间消失，也不必然更快；本表只是给出了这一组教学数值。

`#wrong=0` 表示本次启用的校验未报告错误；测试若未启用校验，相应空值不能读成零错误。还应检查整次运行是否完成、退出状态和测试的错误汇总，防止只截取成功行。这里的 time 是预热后计时得到的单次操作时间，不含整个启动、初始化和扫描的总时长；多 Rank 的报告还应连同平均/最大值等报告设置理解。

该版 `PERFORMANCE.md` 开头将时间写作 ms，但 `common.cu` 实际将秒数乘 `10^6` 后输出，字段为微秒。本表按输出表头与代码的 `us` 解读，复算时换成秒；不能直接套用说明页开头的单位。核对位置见 [common.cu 的 TimeTest 与结果输出](https://github.com/NVIDIA/nccl-tests/blob/v2.19.7/src/common.cu)。

## 先算 algbw，再算 busbw

令 t 为该次操作的报告时间，M 为每 Rank 的 AllReduce 输入字节数：

```text
algbw = M / t
AllReduce busbw = algbw × 2(N−1)/N
```

N=4 时系数为 1.5，与第二篇 Ring 每 Rank 发 1.5M 的结果一致。这个系数由测试按**集合类型与 Rank 数**计算，报告 Tree 时也会使用它；看到 1.5 倍关系不能反推测试采用 Ring。换算直接见 [all_reduce.cu：AllReduceGetBw](https://github.com/NVIDIA/nccl-tests/blob/v2.19.7/src/all_reduce.cu)。

用最后一行左侧数据计算：

```text
M = 256 × 2^20 = 268435456 B
t = 25000 μs = 0.025 s
algbw = 268435456 / 0.025 / 10^9 = 10.73741824 GB/s
busbw = 10.73741824 × 1.5 = 16.10612736 GB/s
```

结果四舍五入为 10.74 和 16.11。不要用已舍入的 10.74 再要求 busbw 精确到所有小数位。也不要把 M 先乘四：输出已采用每 Rank 的消息定义，全组物理发送量是另一个问题。

对其他集合，必须先统一大小 S 的含义，再使用对应系数：

| 操作 | 此处 S 的口径 | busbw / algbw |
| --- | --- | --- |
| AllReduce | 每 Rank 完整输入 | `2(N−1)/N` |
| AllGather | 每 Rank 收齐后的总数组，等于 N 倍本地输入 | `(N−1)/N` |
| ReduceScatter | 每 Rank 完整输入，等于 N 倍本地输出 | `(N−1)/N` |
| Broadcast / Reduce | 根相关完整数组 | `1` |

例如四人 AllGather 每人先有 64 MiB，收齐 256 MiB，t=20 ms，则以 S=256 MiB 算 `algbw≈13.42 GB/s`、`busbw≈10.07 GB/s`，不能先用 64 MiB 再套同一公式。口径和公式见 [v2.19.7 PERFORMANCE：Algorithm bandwidth / 各集合小节](https://github.com/NVIDIA/nccl-tests/blob/v2.19.7/doc/PERFORMANCE.md)。其他测试操作应读取其对应定义，避免仅凭表头 `size` 猜参数意义。

## busbw 何时能和 NIC 比较

busbw 是按通信量模型归一化的性能指标，不是从 NIC、PCIe 或 NVLink 计数器采到的物理吞吐。它用于比较时，仍要确认消息、规模、拓扑和算法对应的流量究竟落在哪里。

在第三篇**指定单环、无卸载、两机各两卡**的场景中，每台 NIC 每方向确实承载 1.5M；若 t=25 ms，那么 `1.5M/t≈16.11 GB/s` 正好也是该端口在整个操作时间内的平均发送有效负载速率。本题给定单向有效预算 20 GB/s，16.11/20≈80.53%，只能表示相对这个教学预算的负载速率比例；固定开销、归约、间歇和共享资源都可能解释差距，不能据此判定线速利用率或某个组件效率。

换成单机四卡，busbw 主要反映本机互联；换成每 GPU 一张 NIC、多通道或分层集合，整机 NIC 汇总与每 Rank 指标之间的关系又会变。不同规模下系数也变，例如 N=8 的 AllReduce 系数是 1.75。因此记录时至少带上集合类型、N、每节点卡数、消息口径、布局和实际网络路径。

## 从单链路到集合，逐步扩大验证范围

`perftest` 是通过 Verbs 发起的点对点微基准。`ib_write_bw` 测给定两端、内存与参数下的 RDMA Write 带宽，不执行 NCCL 的四人归约。若默认测试主机内存，不能据此证明显存路径；GPU 模式还依赖构建选项和支持的注册方式。测试类别及限制见 [linux-rdma/perftest：Overview / Benchmarks Description / GPUDirect usage](https://github.com/linux-rdma/perftest/blob/8f8784488da8181c52334c2cff2d87e1bd2ae33d/README)，固定到核对时的 commit `8f8784488da8181c52334c2cff2d87e1bd2ae33d`，不依赖命令行默认值。

| 测试层次 | 新增覆盖 | 条件满足并成功后可说什么 |
| --- | --- | --- |
| 主机内存 perftest | 两端 NIC、选定网络路径与主机 MR | 这对端点的主机内存 RDMA 工作，给定参数下速率已知 |
| GPU 内存 perftest | 指定 GPU—NIC、显存注册与交付 | 记录明确路径与校验后，该对显存端点的传输成立 |
| 单机 nccl-tests | 本机多 GPU、集合和本机路径 | 指定本机组的集合功能/性能成立 |
| 两机各一卡 nccl-tests | 跨机 NCCL 初始化与一对 GPU | 指定 GPU/NIC 组合的跨机集合成立 |
| 两机各两卡及更大组 | NIC 共享、多成员配合、更多路径和并发 | 给定布局和规模的集合完成，得到相应曲线 |

按这个顺序建立材料，是因为每一步都引入新的依赖。单链路正常而四卡慢，不会自动把责任落到 NCCL；两条并发流可能正在共享一个本来正常的端口。四卡通过也不等于大规模跨机架路径、拥塞与尾延迟已经验证。

## 一份可比较结果需要哪些条件

记录测试二进制/源码与实际加载的 NCCL、CUDA/驱动、GPU/NIC/固件、内存类型、节点数与 Rank 映射、NIC/端口、CPU/NUMA 放置、消息扫描、预热/迭代/校验、计时方式和强制参数。性能比较还要固定并发背景、方向与布局，保留多次结果或波动范围，避免只拿某次最好值。

小消息重点看时间及波动，大消息重点看吞吐是否随消息增大进入平台；只测 256 MiB 会漏掉小消息延迟问题，只测两张卡会漏掉共享与同步问题。主机内存 Write 为 20 GB/s、四卡 AllReduce algbw 为 10.74 GB/s，并不能直接得出“集合只有网络一半快”：两者操作、内存、参与者和流量口径都不同。先换算并画出共享路径，才有可解释的比较。

[上一篇：沿阶段阅读 NCCL 日志](04_沿阶段阅读NCCL日志.md) · [下一篇：用两机四卡材料完成综合推演](06_用两机四卡材料完成综合推演.md)
