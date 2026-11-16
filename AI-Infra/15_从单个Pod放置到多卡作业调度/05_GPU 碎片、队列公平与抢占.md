
`train-12` 的一个 worker 要在同一节点凑齐四种资源，三个 worker 还可能要落在同一个网络域。一个大作业的等待时间取决于三个决定。放置策略影响空余资源集中在哪些节点，队列规则影响这些额度先给谁，抢占决定已承诺的资源能否提前释放。

## 装箱怎样改变 GPU 余量的分布

取一组独立的教学节点。三台节点各有八张同型 GPU，CPU、内存、网卡和其他放置条件都充足。六个先到的小 Pod 每个申请一张卡。随后到达的作业需要两个 worker，每个要在一台节点获得八张卡。

| 六个小 Pod 的放置 | 节点 A 剩余 | 节点 B 剩余 | 节点 C 剩余 | 余量合计 | 八卡落点数 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 每台放两个 | 6 | 6 | 6 | 18 | 0 |
| 集中在节点 A | 2 | 8 | 8 | 18 | 2 |

第一行每台机器都只差两张卡，三台的余量却无法合成一个八卡落点。这就是相对于八卡请求的**资源碎片**。第二行把小请求集中到一台机器，留出两台完整的节点。把小请求集中到较满的合格节点的思路称为**装箱**（Bin Packing）。若两处八卡落点分属不同网络域，而新作业要求同域，仍需按域重算，任何一侧都只能放一个。

Kubernetes v1.36 的 `NodeResourcesFit` 可配置 `MostAllocated`，按放入当前 Pod 后指定资源的分配比例给较满的候选节点更高分。比如已有小 Pod 占用节点 A 的一张卡，第二个小 Pod 放到 A，GPU 分配比例为 `2/8`，放到空闲节点 B 为 `1/8`。以下是 `KubeSchedulerConfiguration` 中单个 profile 的教学摘录，仅指定这个插件使用的 GPU 评分维度。硬性过滤仍先执行，最终选点还取决于其他启用的评分插件。 [Kubernetes v1.36 · 资源装箱评分](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/) · [Kubernetes v1.36 · 调度框架](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)

```yaml
pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated
        resources:
          - name: nvidia.com/gpu
            weight: 1
```

## 队首规则与额度累积

回到额度准入。某个大作业先到，需要 12 个同类 GPU 额度。其后一个同优先级的小作业需要 4 个。起初队列余量为 8，稍后已有作业释放 4 个额度。假定没有借用或抢占，其他准入条件均通过。在小作业先跑的路径中，假定它在 4 个额度释放时仍在运行。Kueue v0.17 的两种队列策略可以产生下面的结果。 [Kueue v0.17 · ClusterQueue 队列策略](https://kueue.sigs.k8s.io/v0.17/docs/concepts/cluster_queue/#queueing-strategy)

| 策略 | 初始有 8 个额度 | 已有作业释放 4 个后 |
| --- | --- | --- |
| `StrictFIFO`，严格按队首处理 | 大作业在队首等，其后的小作业也等。 | `8 + 4 = 12`，可以为大作业保留额度。 |
| `BestEffortFIFO`，允许符合条件的后续作业先行 | 小作业使用 4 个，剩余 4 个。 | `4 + 4 = 8`，大作业继续等。 |

首行在等待期间保留了 8 个闲置额度。次行让其中 4 个先产生工作，但大作业要等下一次释放。如果小作业在已有作业释放额度前已经结束，两条路径在那个时点都能凑足 12。比较队列策略时要同时看小作业的运行时长和大作业的资源缺口。

## 队列之间的借用与公平份额

同一队列内的先后顺序确定后，多租户还要解决不同队列怎样共享额度。Kueue 的 **ClusterQueue** 维护不同资源规格可用的额度，**cohort** 将多个队列归入可共享额度的组。若甲队列的名义额度为 8、已用 8，乙队列有 4 个同规格的未用额度，在借用和出借限制允许时，甲可以从组内再借 4 个。乙有新需求时，能否收回已借出的额度取决于回收策略和在跑作业。 [Kueue v0.17 · ClusterQueue 与借用](https://kueue.sigs.k8s.io/v0.17/docs/concepts/cluster_queue/#cohort)

启用以抢占为基础的 **Fair Sharing** 时，Kueue 用带权份额比较同组队列占用可借资源的程度，准入时倾向于先处理份额较低的队列，再按配置决定能否通过作业抢占回收额度。如果甲、乙的作业都可以用下一批额度，且其他条件相当，较低份额的队列可以先取得这批额度。公平排序改变的是队列间的获得机会，之后仍要核对作业在物理节点上的落点。 [Kueue v0.17 · Fair Sharing 的加权份额与抢占](https://kueue.sigs.k8s.io/v0.17/docs/concepts/preemption/#fair-sharing)

## 优先级、抢占与有效落点

在常规的 Kubernetes Pod 调度中，较高的 Pod 优先级可影响排队顺序。**抢占**允许调度器在找不到可用节点时，检查移除某台候选节点的低优先级 Pod 后能否放下这个 Pod。配置 `preemptionPolicy: Never` 的 Pod 仍可按优先级排队，并等待资源释放。即使出现 `status.nominatedNodeName`，也要等受影响的 Pod 退出并重新检查，再由 `spec.nodeName` 确认最终绑定。Kueue 的作业级抢占按队列规则回收已准入作业的额度，是否因此出现所需物理落点取决于实际释放位置。 [Kubernetes v1.36 · Pod 优先级与抢占](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) · [Kueue v0.17 · 作业抢占](https://kueue.sigs.k8s.io/v0.17/docs/concepts/preemption/)

取第 03 篇的网络域快照，`train-12` 要求同域。假定可以选择让一个低优先级任务退出，两种候选都释放两张 H100，其他条件保持不变。这里比较受害任务退出后的账本。

| 释放位置 | 对 A 域的影响 | 整组结果 |
| --- | --- | --- |
| `h-a3` | GPU 从 2 增为 4，CPU、内存和 VF 已足够，新增 1 个落点。 | A 域从 2 个增为 3 个，满足集群侧成组条件。 |
| `h-b1` | B 域的节点更宽裕，A 域仍为 2 个落点。 | 同域的三节点组依然放不下。 |

这个推演假设释放已经完成。具体实现还须支持按所需约束选择受害者。例如 Kubernetes v1.36 的原生 Topology-Aware Workload Scheduling 不因同域需求主动触发抢占。单个 Pod 的提名也只能当作一次候选决策，零散提名能否组成同域的一组还要另算。 [Kubernetes v1.36 · 原生拓扑放置的抢占条件](https://v1-36.docs.kubernetes.io/docs/concepts/workloads/workload-api/topology-aware-scheduling/) · [Kubernetes v1.36 · 提名与绑定](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)

若 `h-a3` 上被移除的任务从上次可恢复的 **checkpoint**（训练检查点）后已用两张 GPU 各计算 20 分钟，且这段进度要全部重做，则这台节点的重做量为 `2 × 20 = 40 GPU·min`。它只计受影响节点上这一段已投入的 GPU 时间。实际恢复耗时还受退出宽限期、检查点读取和其他 Rank 是否重启影响。第 16 章会将这些成本接到作业重试与资源释放过程。 [PyTorch 2.9 · 故障、成员变化与检查点](https://docs.pytorch.org/docs/2.9/elastic/run.html#important-notices)
