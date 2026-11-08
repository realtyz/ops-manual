# 沿阶段阅读 NCCL 日志

## 先收齐成员，再把字段放回阶段

一份有用记录应能回答：这是哪次作业、哪个通信组、哪个成员，当前完成到哪一步？只搜索最后一个 `WARN`，容易漏掉先退出的成员；只留 r0，也可能漏掉某一侧的注册或接收错误。

下面是**用于理解采集意图的 Bash 片段**，不是本章要求运行的实验。假设目录已存在且可写，启动器会将变量传给所有 Worker；环境传播方式随启动器核对。

```bash
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=INIT,BOOTSTRAP,NET,GRAPH,P2P,SHM,REG
export NCCL_DEBUG_FILE=/tmp/job-08/nccl.%h.%p.log
```

`%h`、`%p` 分别用主机名和 PID 区分文件，避免成员覆盖彼此。还应保存启动参数、应用调用/阶段事件、加载的 NCCL 版本及退出状态；`INFO` 并不承诺记录每次集合操作的全部参数。深查调用次序时，可针对短复现采用对应版本的调用跟踪，并与应用事件对照。日志级别和子系统见 [NCCL 2.31.2 Logging](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/troubleshooting/logging.html)。

接下来的片段均为**依照 2.31.2 字段自拟、删节的教学日志**，省略线程号等无关前缀或后缀的地方用 `...` 表示；省略不表示该步成功。IP 使用文档示例地址：管理侧 A/B 为 `192.0.2.11/12`，RoCE 侧为 `198.51.100.11/12`，不供部署使用。

## 身份：方括号不是全局 Rank

```text
nccl-a:4100:4110 [0] NCCL INFO ncclCommInitRank comm 0x1000 rank 0 nranks 4 cudaDev 0 nvmlDev 0 busId 41000 commId 0x808 - Init COMPLETE
nccl-a:4101:4111 [1] NCCL INFO ncclCommInitRank comm 0x2000 rank 1 nranks 4 cudaDev 1 nvmlDev 1 busId 42000 commId 0x808 - Init COMPLETE
nccl-b:5200:5210 [0] NCCL INFO ncclCommInitRank comm 0x3000 rank 2 nranks 4 cudaDev 0 nvmlDev 0 busId 41000 commId 0x808 - Init COMPLETE
nccl-b:5201:5211 [1] NCCL INFO ncclCommInitRank comm 0x4000 rank 3 nranks 4 cudaDev 1 nvmlDev 1 busId 42000 commId 0x808 - Init COMPLETE
```

以第三行为例，主机 `nccl-b`、PID 5200 对应 r2；前缀 `[0]` 是该日志的本地 CUDA 设备上下文，不能读成 r0。`rank 2 nranks 4` 才建立 W 中成员身份。`cudaDev 0` 和 `nvmlDev 0` 在本例相同；设备可见性变更后要按第一篇重新映射。

`busId 41000` 是此输出采用的紧凑十六进制表示，本例对应 `0000:41:00.0`；不要当作十进制 GPU 编号。两个主机可以有相同 busId。`comm 0x...` 是本进程句柄地址，跨进程不同很正常；`commId` 可辅助关联同一通信器，但仍需作业和成员上下文。初始化字段参考 [v2.31.2-1 init.cc 中的 Init COMPLETE 输出](https://github.com/NVIDIA/nccl/blob/v2.31.2-1/src/init.cc)。

四行共同证明四个本地通信器都报告初始化完成；还没有提供一次集合操作的完成和结果校验。若只拿到其中三行，先判断是文件缺失、成员未到达还是失败退出，不能把缺行直接写成“第四张 GPU 坏了”。

## Bootstrap：协调接口与负载接口可以不同

```text
nccl-a:4100:4110 [0] NCCL INFO Bootstrap: Using eno1:192.0.2.11<0>
nccl-b:5200:5210 [0] NCCL INFO Bootstrap: Using eno1:192.0.2.12<0>
nccl-a:4100:4110 [0] NCCL INFO Using network IB
nccl-b:5200:5210 [0] NCCL INFO Using network IB
```

前两行说明本次早期 IP 协调选了 `eno1`；后两行说明通信器选择名为 IB 的网络实现，两件事可以同时成立。不要把 Bootstrap 行解释为“所有梯度都走管理网”。该格式可对照 [bootstrap.cc 的接口选择输出](https://github.com/NVIDIA/nccl/blob/v2.31.2-1/src/bootstrap.cc)。

再补充一份**作者设计的端口映射摘要**，字段不是 NCCL 固定输出：

| 主机 | NET/IB 设备索引 | RDMA 设备/端口 | Link Layer | 以太网接口 / IP |
| --- | --- | --- | --- | --- |
| nccl-a | 0 | mlx5_0 / 1 | Ethernet | ens6f0np0 / 198.51.100.11 |
| nccl-b | 0 | mlx5_0 / 1 | Ethernet | ens6f0np0 / 198.51.100.12 |

结合题设 RoCEv2 与 `Ethernet`，才得到该 IB 传输在使用 RoCE 的判断。设备索引 0 是 NET 实现内部索引；不能由数字 0 自动推出 HCA 名 `mlx5_0`，本题是通过映射摘要给出对应。

## Channel：看逻辑端点、方向和本地传输

```text
nccl-a:4100:4110 [0] NCCL INFO Channel 00/0 : 0[0] -> 1[1] via P2P/CUMEM
nccl-b:5200:5210 [0] NCCL INFO Channel 00/0 : 2[0] -> 3[1] via P2P/CUMEM
nccl-a:4101:4111 [1] NCCL INFO Channel 00/0 : 1[1] -> 2[0] [send] via NET/IB/0/GDRDMA(PCI)...
nccl-b:5200:5210 [0] NCCL INFO Channel 00/0 : 1[1] -> 2[0] [receive] via NET/IB/0/GDRDMA(PCI)...
nccl-b:5201:5211 [1] NCCL INFO Channel 00/0 : 3[1] -> 0[0] [send] via NET/IB/0/GDRDMA(PCI)...
nccl-a:4100:4110 [0] NCCL INFO Channel 00/0 : 3[1] -> 0[0] [receive] via NET/IB/0/GDRDMA(PCI)...
```

`00/0` 分别包含通道编号和连接索引，不表示 NIC 的端口 0。`1[1] -> 2[0]` 中，外面的 1、2 是通信 Rank，里面的 1、0 是该版这类 Channel 行记录的 NVML 设备索引。不要用前缀方括号的解释机械替换行内方括号。字段来源见 [p2p.cc](https://github.com/NVIDIA/nccl/blob/v2.31.2-1/src/transport/p2p.cc) 和 [net.cc 的 sendSetup / recvSetup](https://github.com/NVIDIA/nccl/blob/v2.31.2-1/src/transport/net.cc)。

前两行支持本机两条边按 P2P 准备；`CUMEM` 表示相应的内存句柄路径，不表示经过主机 DRAM，也不证明有 NVLink。后四行把跨机边两端对应起来：例如 r1→r2，A 侧报告 send，B 侧报告 receive，两个端点均报告 GDRDMA。`(PCI)` 与本题 GPU—NIC 的 PCIe 路径一致；这里只展示连接准备信息，后缀省去共享/flush 等细节，不能用它编写显存可见性协议。

这六行与第二篇环相容，但仅凭连接表仍不能证明某一次 AllReduce 采用了这条 Ring。连接资源可被后续多种操作使用，初始化也可能准备多种候选图。要确定调用算法，应补该调用对应的调优/跟踪信息；普通 INFO 中存在 Ring 行并不等于所有消息固定用 Ring。

再看两份**彼此独立的路径变体**，不与上面的基线同时成立；这里省略主机前缀：

```text
变体 S：本机边改用主机共享缓冲
Channel 00 : 0[0] -> 1[1] via SHM/direct

变体 T：跨机网络实现被指定为 Socket
Using network Socket
Channel 00/0 : 1[1] -> 2[0] [send] via NET/Socket/0
```

S 的 `SHM/direct` 指向主机共享缓冲传输，`direct` 不能单独读成 GPU 显存直达；应结合完整传输前缀。T 中 `/0` 仍是网络实现的设备索引，需另查实际 IP 接口；它与 Bootstrap 同在 eno1 还是使用另一个接口，不能靠这行决定。S 的格式对照 [shm.cc 的 sendSetup](https://github.com/NVIDIA/nccl/blob/v2.31.2-1/src/transport/shm.cc)。这样可以用相同的“成员边→传输类别→本地设备”顺序读不同路径，而不是凭某个单词判定。

## 错误：将最早失败与随后等待连接起来

下面三份**独立教学材料**不属于同一次运行。

| 材料 | 能确定的层次 | 下一份证据的用途 |
| --- | --- | --- |
| 初始化时 `Call to ibv_reg_mr failed`，随后 API 返回系统错误 | 内存注册调用失败 | 对照失败缓冲类型、参数、errno、锁页额度和驱动，分清资源限制与映射不支持 |
| 四人已 Init COMPLETE；r0—r2 进入第 9 次 AllReduce；r3 的应用记录为第 9 次 Broadcast | 调用不匹配 | 直接修正分支/次序；网络重试不能让两个操作变成同一操作 |
| r3 先退出，其他成员稍后报远端错误或超时 | 其他成员可能是后续受影响者 | 查看 r3 退出前的本地应用/CUDA/系统事件，定位最早失败 |

MR 注册失败不能直接等同于“网络不通”或“肯定是 memlock”。同样，一个 timeout 只是观察到等待超过阈值；缺成员、错序、设备执行问题和网络故障都可能造成等待。排查先沿成员和阶段缩小范围，再选能够验证该层假设的材料。错误分类参考 [Runtime and MPI issues：Errors](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/troubleshooting/runtime_and_mpi_issues.html#errors)。

## 配置用于回答具体问题

本章只保留几种有明确对象的设置，语义按 [Environment Variables](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/env.html) 各同名小节核对：

| 设置示意 | 指定的对象 |
| --- | --- |
| `NCCL_SOCKET_IFNAME='=eno1'` | 精确匹配 IP 接口 |
| `NCCL_IB_HCA='=mlx5_0:1'` | 精确匹配 RDMA 设备及端口 |
| `NCCL_NET=Socket` | 指定网络实现，用于受控对照 |
| `NCCL_ALGO=Ring` | 限定候选算法，用于验证算法差异 |

前两项分别作用于接口和 HCA，不能互相替代。更改任一项后，都要核对所有成员收到的实际值与新日志。对照实验记录原设置，只改一个条件，并同时观察正确性和耗时；例如 Socket 对照成功，只将范围缩小到路径差异，还需要 GPU 注册、RDMA 单链路等证据定位原因。强制参数改变了自动选择条件，对照完成后应恢复正常选择并复测目标规模。

[上一篇：把逻辑通信放回设备路径](03_把逻辑通信放回设备路径.md) · [下一篇：读懂 nccl-tests 与分层测试](05_读懂nccl-tests与分层测试.md)
