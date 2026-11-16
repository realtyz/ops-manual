
`train-12` 需要三个四卡 worker，并且每台机器最多一个。先假定队列和节点都允许放置多个作业。此时调度器还要决定哪个队列、哪个作业先得到分配机会，以及合格的节点中优先用哪台。以下独立快照沿用 Volcano v1.16.0、`capacity` 插件和平级队列。节点 `h1/h2/h3` 各有 8 GPU 和 32 CPU，除了算例明确讨论的资源，其余条件均满足。

## Capacity 的上限、应得份额和专属保留

`train` 在用 4 GPU 时可以再接 12 GPU，因为其上限为 16。若它最终占到 16 GPU，在两个队列都需要资源时，可以从应得份额算出超额部分。队列的三项字段分别回答三个问题。 [Volcano v1.16.0 Queue 字段](https://volcano.sh/docs/v1.16.0/concepts/queue/) · [Capacity 配置指南](https://volcano.sh/docs/v1.16.0/userguide/user_guide_how_to_use_capacity_plugin/)

| 字段 | 对用量的作用 | 本例 GPU 数 |
| --- | --- | --- |
| `capability` | 队列资源使用上限，还受集群容量和其他队列保留量约束。 | `train` 16，`batch` 24 |
| `deserved` | 应得份额，用于计算相对占用和资源竞争时的借用量。 | 两队列各 12 |
| `guarantee.resource` | 为该队列专属保留的总量。只要保留仍配置，其余队列可使用的容量就要扣除这一部分。 | 主例均为 0 |

例如 `batch` 占 16 GPU 时已借用 `16 − 12 = 4`，全池尚余 `24 − 16 = 8`。若此时空闲的 `train` 需要 12 GPU，总量再缺 4，恰好与 `batch` 的超额相同。是否能收回这 4 张，要看可驱逐的 Pod 和卡所在的节点。

将资源池清空，另设 `train` 的 `guarantee.resource` 为 4 GPU。对 `batch`，源码先从池容量扣除其他队列的保留，再以自己的 `capability` 封顶，所以 GPU 有效上限为 `min(24, 24 − 4) = 20`。在该平级算例中，`train` 自己仍受 16 的上限限制。这里只算队列级可用数量，8 卡节点能剩多少由已有 Pod 的分布决定。 [v1.16.0 capacity 有效上限](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/plugins/capacity/capacity.go#L997-L1019)

`proportion` 也是队列配额插件。它用队列的 `weight` 和需求从可用资源中动态计算份额。举例来说，两个队列权重 `1:1`、需求都充足且无其他约束时，24 GPU 各分 12，节点池扩到 32 GPU 后各分 16。`capacity` 的两份显式 `deserved: 12` 需要按策略维护。两种插件择一使用。 [Volcano v1.16.0 队列资源管理](https://volcano.sh/docs/v1.16.0/keyfeatures/queueresourcemanagement/) · [Proportion 插件](https://volcano.sh/docs/v1.16.0/scheduler/plugins/proportion/)

## 队列优先级与作业主导份额

对都能参加本轮分配的平级队列，`capacity` 排序回调先比 Queue 的 `spec.priority`，高者在前。若优先级相同，比较已分配量占各自应得份额的比例，对纳入计算的资源维度取最大值。例如 GPU 比例主导时，`train` 用了 4 卡和 `batch` 用了 8 卡，份额分别为 `4/12` 和 `8/12`，优先考虑 `train`。如果其他已配置资源的比例更高，要按最高项重算。 [v1.16.0 capacity 队列排序](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/plugins/capacity/capacity.go)

选到队列以后，作业顺序由另一组回调给出。**DRF**（Dominant Resource Fairness，主导资源公平）按作业已分配的各项资源分别除以集群同项总量，最大的比例是作业主导份额。正在等更多成员的 A、B 在同一队列，集群共有 24 GPU、96 CPU，其他项的占比均不高于下表的最大值。 [Volcano v1.16.0 DRF](https://volcano.sh/docs/v1.16.0/scheduler/plugins/drf/) · [v1.16.0 DRF 作业排序](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/plugins/drf/drf.go)

| 作业已分配 | GPU 份额 | CPU 份额 | 主导份额 |
| --- | ---: | ---: | ---: |
| A，4 GPU、16 CPU | `4/24` | `16/96` | `1/6` |
| B，4 GPU、32 CPU | `4/24` | `32/96` | `1/3` |

在前置回调均打平时，DRF 优先考虑份额较小的 A。若 B 的 Job 指向更高 `value` 的 PriorityClass，且 `priority` 排在 `drf` 前，B 会先被尝试。主例还在 DRF 前配置了 `gang`，同优先级时，若只有一组尚未达到其调度门槛，该排序回调会先考虑这组。排查时分别查 PodGroup 的作业优先级与具体 Pod 自己的优先级。排序影响尝试顺序，能否放下还要查资源与节点条件。 [Volcano v1.16.0 Priority](https://volcano.sh/docs/v1.16.0/scheduler/plugins/priority/) · [v1.16.0 Gang 作业排序](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/plugins/gang/gang.go#L127-L151) · [v1.16.0 回调比较顺序](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/framework/session_plugins.go#L684-L717)

## Binpack 节点评分与作业形状

`allocate` 为一个 Pod 选节点时，先用资源、标签和反亲和排除不合格节点。**Binpack**（装箱）在合格节点中给放入后更满的资源维度更高的分数。两台 8 卡节点当前用量分别为 3 和 0，若新 Pod 请求 1 GPU，放入后 GPU 占用比例分别为 `4/8` 和 `1/8`。启用并正向加权 GPU 维度时，前者在这一维得分较高，其余维度和其他打分插件共同决定实际排序。 [Volcano v1.16.0 Binpack](https://volcano.sh/docs/v1.16.0/scheduler/plugins/binpack/) · [v1.16.0 Binpack 打分实现](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/plugins/binpack/binpack.go#L188-L244)

下面是主例调度配置中 `binpack` 项的变体。外层的 `tiers[].plugins` 省略。`binpack.weight` 调整插件整体分数，`binpack.resources.nvidia.com/gpu` 调整 GPU 维度在其中的权重。

```yaml
- name: binpack
  arguments:
    binpack.weight: 1
    binpack.resources: nvidia.com/gpu
    binpack.resources.nvidia.com/gpu: 2
```

让单卡 Pod 先运行，比较下列假设布局。计算 `train-12` 的落点时，只要某台机器余量达到 4 就记一个落点，反亲和限制每台最多计一次。

| 单卡 Pod 数 | 假设布局 | `h1 / h2 / h3` GPU 占用 | GPU 余量 | 四卡 worker 落点数 |
| ---: | --- | --- | --- | ---: |
| 15 | 分散 | `5 / 5 / 5` | `3 / 3 / 3` | 0 |
| 15 | 紧凑 | `8 / 7 / 0` | `0 / 1 / 8` | 1 |
| 9 | 分散 | `3 / 3 / 3` | `5 / 5 / 5` | 3 |
| 9 | 紧凑 | `8 / 1 / 0` | `0 / 7 / 8` | 2 |

前两行总余量均为 9，紧凑布局还空出了一台完整的八卡节点，适合一个请求 8 GPU 的 Pod。后两行总余量均为 15，紧凑布局却少一台可给 `train-12` 使用的节点。调权重之前，先把常见的大作业换算为每节点请求和所需节点数。节点评分影响以后放进去的 Pod，已经存在的分布要在成员退出或另行迁移后才会改变。

![九个单卡 Pod 分散放置时三台节点各余 5 GPU，各有一个四卡落点。紧凑放置时三台节点分别余 0、7、8 GPU，只剩两个落点。两种布局总余量都是 15 GPU。](images/fig-16-03-01-binpack-worker-fit.png)

图 16-03-01　九个单卡 Pod 的两种教学布局。括线表示同一节点上的一个四卡落点，图例「可分配」指扣除已有请求的余量。这里假定 CPU、内存和 VF 也足够。若紧凑布局中的 `h1` 能释放任意四卡且配套资源满足，落点又变为三个。下一篇用该思路判断驱逐候选。
