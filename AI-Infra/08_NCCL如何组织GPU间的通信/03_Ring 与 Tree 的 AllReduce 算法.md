# Ring 与 Tree 的 AllReduce 算法

同样是四张 GPU 做一次 AllReduce，为什么消息只有几 KiB 时树形（Tree）能快出一倍，而一旦消息涨到几十 MiB，环形（Ring）反而稳稳反超？

这道分水岭恰恰把两类核心 AI 负载劈成了两条截然不同的路线：**分布式训练的大梯度同步**追求把整条链路的带宽吃满，而**在线推理的张量并行（TP）小消息**则寸土必争地抢夺微秒级延迟。要真正理解 NCCL 的算法决策，我们先在逻辑拓扑上把 Ring 与 Tree 的每一轮搬运跟踪透彻。

## Ring：切块流水让每条边同时满载

假设四个成员连成一个逻辑单向环 `0 → 1 → 2 → 3 → 0`，每张卡持有 `4 MiB` 的输入张量。如果整块 `4 MiB` 一股脑沿环传递，同一时刻只有一条边在忙，其余三张卡都在干等。

Ring 算法的巧妙之处在于**按数组位置把每张卡的输入切成与成员数相同的 4 个 `1 MiB` 数据块（`C0` 至 `C3`）**，让不同数据块在第一轮从不同成员同步出发，把四条环边同时铺满。整个 AllReduce 自然拆成第 01 篇提到的两个阶段：

| 阶段与轮次 | `0 → 1` | `1 → 2` | `2 → 3` | `3 → 0` |
| --- | :---: | :---: | :---: | :---: |
| ReduceScatter 第 1 轮 | **C0** | C1 | C2 | C3 |
| ReduceScatter 第 2 轮 | C3 | **C0** | C1 | C2 |
| ReduceScatter 第 3 轮 | C2 | C3 | **C0** | C1 |
| AllGather 第 1 轮 | C1 | C2 | C3 | **C0** |
| AllGather 第 2 轮 | **C0** | C1 | C2 | C3 |
| AllGather 第 3 轮 | C3 | **C0** | C1 | C2 |

沿着加粗的 **`C0`** 走一遍全程：
1. **ReduceScatter 阶段（3 轮归约）**：第 1 轮 Rank 0 发出本地 `C0`，Rank 1 收到并加上自己的 `C0`；第 2 轮传给 Rank 2，第 3 轮抵达 Rank 3 合成全局完整的 `C0`。
   - 同期 `C1`、`C2`、`C3` 也分别在 Rank 0、Rank 1、Rank 2 上完成归约（注：这是 AllReduce 内部分片落点；独立 `ncclReduceScatter` 仍按组内 Rank 顺序交付）。
2. **AllGather 阶段（3 轮分发）**：Rank 3 把完整的 `C0` 沿环再发 3 跳（`3 → 0 → 1 → 2`），途经的每张卡直接覆盖写回原数组位置，不再做加法。6 轮结束，四张卡全部拿齐完整的 `C0` 至 `C3`。

把这一规律推广到 $P$ 个成员、单卡输入 $S$ 字节的等分块单环：每个阶段固定走 $P - 1$ 轮，每轮每张卡在发送与接收方向各搬运 $S / P$ 字节。不计协议头时，单卡在一次 AllReduce 中的算法发送量与接收量相等：

$$V_{\text{send}} = V_{\text{recv}} = 2(P - 1) \times \frac{S}{P} = \frac{2(P - 1)}{P} S$$

代入四卡 `4 MiB` 的例子（$P = 4, S = 4\text{ MiB}$），单卡经历 6 轮、单向累计发送 $1.5 \times 4 = 6\text{ MiB}$、接收 $6\text{ MiB}$。哪怕集群扩大到上千张卡，系数 $\frac{2(P-1)}{P}$ 也永远被锁死在 $2$ 以内——这就是 Ring 带宽扩展性极其出色的根本原因。

## Tree：多分支并行缩短跳数

Ring 的短板同样一眼可见：总轮数 $2(P - 1)$ 随成员数线性膨胀。规模一大，哪怕只传几个字节，光是一跳一跳排队启动的固定延迟就足以拖垮全程。

换成树形结构，情况立刻改观。看图 08-03-01 右侧的四卡树：Rank 0 为根，Rank 1、2 为子节点，Rank 3 挂在 Rank 1 下方。依然只追踪 `C0`：
- **归约阶段（2 轮）**：第 1 轮，Rank 3 把贡献发给 Rank 1，同时 Rank 2 把贡献发给根 Rank 0；第 2 轮，Rank 1 把合并后的部分和送给 Rank 0，根随即合成全局 `C0`。
- **广播阶段（2 轮）**：沿反向树边下发，第 1 轮根同时把完整 `C0` 发给 Rank 1 和 Rank 2；第 2 轮 Rank 1 转发给 Rank 3。

![四 Rank 的 Ring 和 Tree 对照。环上 C0 沿 0→1→2→3 归约，再沿 3→0→1→2 分发。树上 C0 先从 3 到 1、从 2 到 0，再从 1 到根 0，最后沿反向树边广播。](images/fig-08-03-01-ring-vs-tree.png)

*图 08-03-01：四成员通信组下 Ring 与 Tree 追踪数据块 `C0` 的路径与轮次对照（橙色为归约传递，绿色为结果分发，假定同轮独立分支并行推进）。*

在平衡树里，最长依赖路径从 Ring 的线性 $O(P)$ 锐减为对数 $O(\log_2 P)$。但传统单树存在明显隐患：叶子节点只收发一次，中间节点却要同时对父子两端转发，很容易形成带宽瓶颈。

为了打破这个瓶颈，NCCL 从 2.4 起引入**双二叉树（Double Binary Trees）**，把数据拆给两棵结构互补的树并行推进。任何成员至多在其中一棵里充当中间节点，在另一棵里则是叶子节点，既保住了对数级低延迟，又把转发带宽压力均匀摊开。

## 八卡延迟-带宽模型：定量看清拐点

用一组干净的八卡（$P = 8$）对称模型把 Ring 和 Tree 的胜负线算出来。假设每跳固定启动开销 $\alpha = 2\,\mu\text{s}$，并行通信对各享单向有效带宽 $B = 25\text{ GB/s}$，收发全双工重叠，暂不计协议头、归约计算与共享争用：

- **分块 Ring**：共 $2(P - 1) = 14$ 轮，每轮传 $S / P$ 字节：

$$T_{\text{ring}}(S) = 2(P - 1)\left(\alpha + \frac{S}{P \cdot B}\right) = 14\alpha + \frac{1.75\,S}{B}$$

- **无分块二项树**：每阶段 $\log_2 P = 3$ 轮、共 $6$ 轮，关键路径每轮传完整 $S$ 字节：

$$T_{\text{tree}}(S) = 2\log_2 P\left(\alpha + \frac{S}{B}\right) = 6\alpha + \frac{6\,S}{B}$$

将小消息 `4 KiB`（$4{,}096\text{ B}$）与大消息 `4 MiB`（$4{,}194{,}304\text{ B}$）分别代入，结果高下立判：

| 算法模型 | 耗时公式（$P=8$） | `4 KiB` 耗时 | `4 MiB` 耗时 |
| --- | --- | ---: | ---: |
| **分块 Ring** | $14\alpha + 1.75\,S/B$ | `28.29 µs` | `321.60 µs` |
| **无分块二项树** | $6\alpha + 6\,S/B$ | `12.98 µs` | `1018.63 µs` |

在 `4 KiB` 一列，Ring 仅 14 轮固定延迟就耗掉 `28 µs`，传数据本身只花 `0.29 µs`；Tree 凭着区区 6 轮把总耗时压到不到 Ring 的一半。换到 `4 MiB` 时，Ring 关键路径只搬运 $1.75S$，无分块 Tree 却搬运 $6S$，带宽彻底接管了主导权。令两式相等即可解出理论交点：

$$14\alpha + \frac{1.75\,S^*}{B} = 6\alpha + \frac{6\,S^*}{B} \implies 8\alpha = \frac{4.25\,S^*}{B}$$

$$S^* = \frac{8 \times 2 \times 10^{-6}\text{ s} \times 25 \times 10^9\text{ B/s}}{4.25} \approx 94.1\text{ KB} \approx 92\text{ KiB}$$

> [!summary] 核心结论
> 固定开销 $\alpha$ 越高、节点规模 $P$ 越大，Ring 与 Tree 的交点 $S^*$ 就越往大消息方向移动。真实 NCCL 的双二叉树还会进一步做分块流水，在大规模集群上往往在数 MiB 甚至更高区间依然能与 Ring 抗衡。

## 算法、协议与通道：训练大梯度 vs 推理 TP 小消息

在 NCCL 内部，确定“用 Ring 还是用 Tree”只完成了第一层决策。一次集合通信落到硬件上，由四层机制协同决定：

1. **算法（Algo）**：规定逻辑拓扑与归约顺序，如 `RING`、`TREE`，以及在支持硬件在网归约的 NVSwitch 域或集合网络上启用的 `NVLS`（NVLink SHARP）与 `COLLNET`。
2. **通道（Channel）**：把一次大通信拆成多条并行逻辑管线，交由多组 GPU SM 与多条物理链路同时推进以压榨总带宽。
3. **协议（Proto）**：决定单步数据怎样分片、同步和通知对端就绪，直接权衡微秒级延迟与有效带宽利用率。
4. **传输实现（Transport）**：把每条逻辑边落到真实的机内 P2P/SHM 或跨机 RDMA/Socket 通道（下一篇展开）。

NCCL 内置的三种协议在延迟与带宽之间划出了清晰的梯度：

| 协议 | 同步与封装机制 | 延迟特征 | 有效带宽利用率 | 典型适用场景 |
| --- | --- | --- | ---: | --- |
| **`LL`（Low Latency）** | 每 4 B 数据内联 4 B 同步标志，接收端轮询标志位，省去重型内存栅栏 | 极低（微秒级起步） | 约 50%（标志占一半） | 极小消息（几字节到几十 KiB） |
| **`LL128`** | 依托硬件 128 B 原子写，每 128 B 内放 120 B 数据 + 8 B 标志 | 很低 | 约 95% | 机内 NVLink 等支持 128 B 原子写的中小消息 |
| **`Simple`** | 大块环形缓冲 + 系统级内存栅栏同步 | 较高 | 接近 100% 线速 | 大消息高吞吐传输 |

把算法与协议拼在一起，**训练与推理两类核心负载的通信偏好就完全对上了号**：

| 负载场景 | 典型单次集合消息尺度 | 核心优化目标 | NCCL 典型算法与协议偏好 |
| --- | --- | --- | --- |
| **分布式训练（DP / FSDP 梯度同步）** | 数十 MiB 到数百 MiB（经梯度分桶合并） | 带宽吞吐优先，吃满多 Rail 网络与 NVLink 线速，掩盖在反向计算之后 | **`RING`（或大规模多机 `TREE` 流水）+ 多通道 + `SIMPLE`** |
| **在线大模型推理（TP 张量并行）** | 几 KiB 到数 MiB（Decode 阶段单步隐藏态极小，每层触发两次 AllReduce） | 极致低尾延迟优先，直接决定每 token 生成时间（TPOT）能否守住 SLO | **`TREE` / 机内 NVLink 全直连（或 `NVLS`）+ `LL` / `LL128`** |

> [!tip] 工程直觉：为什么推理 TP 宁可牺牲带宽利用率也要抢 `LL` / `LL128`？
> 在大模型 Decode 阶段，假设批量 16、隐藏维度 8192、精度 BF16，单次 TP AllReduce 张量仅 $16 \times 1 \times 8192 \times 2\text{ B} = 256\text{ KiB}$，但 80 层模型每生成一个 token 要连续做 160 次这样的同步！
>
> 每次省下 `15 µs` 同步延迟，单 token 就能抢回 `2.4 ms`。此时宁可在协议里多花一点标志开销，也绝不能容忍内存栅栏和多跳串行把计算管线冻住。

## 诊断实操：亲眼核对 NCCL 选了什么组合

在 NCCL 2.27.3 中，内部代价模型（`tuning.cc` / `enqueue.cc`）会自动结合操作类型、消息字节数、硬件拓扑与可用能力选出耗时最低的组合。需要做性能对比排障时，千万不要盲猜当前跑的是 Ring 还是 Tree——应先留一轮默认自动选择，再短时通过 `NCCL_ALGO` 做受控对照，并开启 `TUNING` 子系统日志在 Rank 0 上核对实际落点：

```bash
# 仅在诊断轮开启 TUNING 日志，固定绑定与消息尺寸查看 Rank 0 决策记录
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=TUNING
# 如需短时强制对比算法，可再追加：export NCCL_ALGO=Ring  或  NCCL_ALGO=Tree
```

对应在 Rank 0 的诊断输出里，可以一眼看出大消息与小消息截然不同的算法、协议与通道分配：

```text
# 推理 TP 常见小消息（64 KiB）：自动选 TREE + LL128 低延迟协议，仅启用少量通道避免分散启动开销
gpu-a:4100:4100 [0] NCCL INFO AllReduce: 65536 Bytes -> Algo TREE proto LL128 channel{Lo..Hi}={0..1} ...
# 训练梯度常见大消息（64 MiB）：自动切换 RING + SIMPLE 高带宽协议，铺开更多并行通道吃满链路
gpu-a:4100:4100 [0] NCCL INFO AllReduce: 67108864 Bytes -> Algo RING proto SIMPLE channel{Lo..Hi}={0..3} ...
```

> [!warning] 调优边界：`NCCL_ALGO` 白名单与协议联动
> 1. 单独固定 `NCCL_ALGO` 而保持协议自动时，你对比的其实是“算法 + 各自自动匹配的协议”组合；如需隔离单变量，应一并核对日志里的 `proto` 与通道范围。
> 2. 在 NCCL 2.27.3 中，如果 `NCCL_ALGO` 白名单排除了 `Ring` 且当前拓扑/操作无法匹配任何其他合法算法，集合调用会直接报错失败；同时在无 NVSwitch 或未配集合网络插件的集群上，强行指定 `NVLS` 或 `CollNet` 同样不会生效。

回到本章的四卡逻辑环：`0 → 1` 与 `1 → 2` 在逻辑模型里各承担完全相同的字节量，可一旦映射回两台物理服务器，前者留在机内总线，后者必须穿过网卡和交换机。下一篇我们就把这些逻辑边真正钉到物理路径上。

> [!info]- 参考资料
> - **AllReduce / ReduceScatter 语义与双二叉树**：[NCCL 2.27.3 Collective Operations](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/usage/collectives.html#allreduce)、[ReduceScatter 输出顺序](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/usage/collectives.html#reducescatter)、[NVIDIA Blog: Massively Scale Deep Learning Training with NCCL 2.4 (Double Binary Trees)](https://developer.nvidia.com/blog/massively-scale-deep-learning-training-nccl-2-4/)
> - **Ring 传输字节量推导**：[nccl-tests v2.17.9 Performance Formulas](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/doc/PERFORMANCE.md#allreduce)
> - **算法、协议、NVLS 与调优源码**：[NCCL 2.27.3 `NCCL_ALGO` / `NCCL_PROTO`](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-algo)、[`NCCL_NVLS_ENABLE`](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-nvls-enable)、[`src/graph/tuning.cc`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/graph/tuning.cc)、[`src/enqueue.cc`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/enqueue.cc)、[`src/enqueue.cc#L748` (TUNING 日志)](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/enqueue.cc#L748)
