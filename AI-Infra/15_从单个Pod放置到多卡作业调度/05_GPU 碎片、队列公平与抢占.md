# GPU 碎片、队列公平与抢占

三台 8 卡 H100 节点上合计空着 **18 张 GPU**，此时一份只需 2 台整机（16 张卡）的多机训练作业进场，却一个 worker 也放不下。

当集群从“静态盘点余量”进入“多团队长周期混跑”时，一份大作业究竟要等 5 分钟还是 5 小时，取决于三道彼此咬合的策略杠杆：**放置时的装箱形状、队列里的先后与借用规则，以及紧张时的抢占落点**。

## GPU 碎片与 Bin Packing 装箱

先看那 18 张空卡为何“看得见、用不上”。三台 8 卡同型节点（CPU、内存与网卡均充足）上先到了 6 个单卡小任务（如开发调试容器或单卡推理副本），随后才到达一份需要 2 台 8 卡整机的训练作业：

| 6 个单卡任务的放置方式 | 节点 A 余量 | 节点 B 余量 | 节点 C 余量 | 总余量 | 可用 8 卡落点数 |
| --- | ---: | ---: | ---: | ---: | ---: |
| **均匀分散**（每台放 2 个） | 6 | 6 | 6 | 18 | **0** |
| **装箱集中**（全挤在节点 A） | 2 | 8 | 8 | 18 | **2** |

第一行里每台机器都空着 6 张卡，只差 2 张就满整机，三台加起来的 18 张卡却拼不出哪怕 1 个 8 卡落点——这就是相对于大卡数请求的**资源碎片**。

第二行把小请求主动往较满的合格节点里塞，专门腾出完整的空机器留给整机大作业，这种策略称为 **Bin Packing（装箱）**。如果腾出的两台整机分属不同网络域而大作业又要求同域，仍须按域重新核算，单域最多只能拿到 1 个落点。

在 `Kubernetes v1.36` 中，可以通过配置调度插件 `NodeResourcesFit` 的 `MostAllocated` 策略实现 GPU 装箱。例如节点 A 已占 1 张卡时，下一个单卡 Pod 落在 A 的分配率为 $2/8 = 25\%$，落在空节点 B 仅为 $1/8 = 12.5\%$，`MostAllocated` 会给节点 A 更高评分：

```yaml
# KubeSchedulerConfiguration 单个 profile 启用 GPU 装箱评分片段
pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated         # 资源已分配比例越高，节点得分越高
        resources:
          - name: nvidia.com/gpu
            weight: 1               # 聚焦 GPU 维度执行紧凑装箱
```

> [!warning] 训推双轨张力：装箱利好训练，却可能误伤推理高可用
> `MostAllocated` 是保护整机训练槽位的利器，但如果在**在线推理池**里无脑套用装箱，会把同一模型的所有副本全塞进一台机器——一旦该机宕机，线上服务瞬间全灭！工程上往往通过**分池策略**或**多调度器 profile**把二者隔开：离线任务与潮汐训练池启用 Binpack，核心在线推理副本叠加节点反亲和与 `topologySpreadConstraints` 兜底容灾。

## 队首规则：`StrictFIFO` vs `BestEffortFIFO`

解决了物理余量的形状，再看队列内部“大作业先到、小作业后到”的经典矛盾。

假设队列当前可用额度为 8 张卡（无借用与抢占），先到的大作业需要 12 张卡，紧随其后的同优先级小作业需要 4 张卡；片刻后已有作业释放 4 张卡，且允许插队的小作业此时尚未结束。`Kueue v0.17` 提供的两种排队策略会走出截然不同的时间线：

| 队列策略 | 初始状态（可用 8 卡） | 已有作业释放 4 卡后 | 权衡取舍 |
| --- | --- | --- | --- |
| **`StrictFIFO`**（严格队首） | 大作业在队首等待，后到小作业一并堵住 | $8 + 4 = 12$，大作业立刻凑齐准入 | 杜绝大作业饿死，但等待期间 8 张卡闲置 |
| **`BestEffortFIFO`**（尽力插队） | 小作业先拿走 4 卡开跑，余量降为 4 | $4 + 4 = 8 < 12$，大作业继续等待 | 短期利用率更高，但若短作业络绎不绝，大作业易被长期饿死 |

当然，如果插队的小作业在别人释放 4 卡之前已经跑完归还，两条路径最终都能按时凑齐 12 张卡。选型时必须结合业务侧小作业的平均时长分布与大作业的资源缺口，而不能孤立评判。

## 跨队列借用与加权 Fair Sharing

同一个 GPU 集群往往服务多个业务部门，每个部门绑定独立的 `ClusterQueue`（集群队列），再通过 **cohort（共享池）** 把多个队列连成可互借空闲额度的资源组。

假设研发甲队列名义额度为 8 且已用满，研发乙队列名义额度为 8 但只用了 4。在借用与出借上限允许的前提下，甲可以顺手借走乙闲置的 4 张卡跑任务，避免昂贵的 H100 空转；等乙提交新任务时，再按回收策略决定是否把借出去的额度要回来。

为了防止某个高产队列无限吞并 cohort 里的公共余量，Kueue 引入了基于抢占的 **Fair Sharing（公平份额）** 机制：按带权份额度量各队列对可借资源的占用程度，准入时优先把下一批额度给**当前加权份额更低**的队列，并在策略允许时通过抢占收回超占份额。

> [!note] 额度公平不等于落点就绪
> 公平份额改变的是不同队列获得配额的先后机会。即便低份额队列如愿拿到 12 张卡的准入额度，进入调度阶段后仍须接受节点多维账本、反亲和与网络域拓扑的硬检验。

## 优先级、抢占有效性与检查点重做代价

当高优先级任务（例如白昼高峰紧急扩容的在线推理副本，或临近截止日期的关键训练任务）找不到可用落点时，**抢占（Preemption）**允许调度器驱逐低优先级 Pod 腾出位置。

在 `Kubernetes v1.36` 常规调度中，高优先级 Pod 若配置了 `preemptionPolicy: Never`，它只享有插队排在调度队列前列的权利，但不会踢掉任何在跑任务；允许抢占时，调度器选中受害节点后会先写入 `status.nominatedNodeName` 作为提名，必须等受害者彻底退出、释放设备并经重新校验后，才写入 `spec.nodeName` 完成最终绑定。

**抢占最容易踩的坑，是“踢掉了卡，却踢错了地方”**。回到第 03 篇的网络域账本，硬性同域的 `train-12` 缺 1 个 A 域落点，现在有两个可驱逐的低优先级任务，各自占着 2 张 H100：

| 被抢占任务所在节点 | 释放 2 张 H100 后的局部变化 | 对同域 `train-12` 的实际效果 |
| --- | --- | --- |
| **`h-a3`（A 域）** | GPU 从 2 增至 4，CPU/内存/VF 原本充足，A 域新增第 3 个落点 | **有效抢占**：A 域凑足 3 处落点，满足成组放置 |
| **`h-b1`（B 域）** | B 域节点空得更多，但 A 域依然只有 2 个落点 | **无效抢占**：白白打断在跑任务，大作业照样卡死 |

此外，具体调度器是否具备“感知网络域的定向抢占”能力也必须核实——例如 `Kubernetes v1.36` 原生 Topology-Aware Workload Scheduling 目前并不因同域约束主动触发抢占，单 Pod 级的零散提名也无法保证拼出同域整组。

每次抢占训练任务，都会产生真金白银的**算力重做代价**。如果 `h-a3` 上被驱逐的训练任务距离上一次已落盘的可恢复检查点（Checkpoint），已经用 2 张 GPU 各自算了 20 分钟且进度无法保留，单就这台节点损失的纯 GPU 计算量就是：

$$
W_{\text{rework}} = 2\text{ GPU} \times 20\text{ min} = 40\text{ GPU}\cdot\text{min}
$$

真实恢复耗时还要叠加进程优雅退出宽限期、重新排队等待、从存储加载检查点以及同组其他 Rank 重启的连锁开销。因此，**尽量把高频检查点机制与低优先级可抢占队列绑定**，才能把抢占代价压到可控范围（第 16 章会结合 Volcano 深入拆解成员重试与恢复链路）。

> [!summary] 核心结论
> 碎片装箱决定“大作业有没有整块地”，队列 FIFO 与 Fair Sharing 决定“谁先拿到入场券”，精准抢占决定“紧急时刻去哪儿腾位置”。当这三把杠杆与**在线大模型推理的昼夜流量波峰波谷**结合起来，就构成了下一篇的主角——在线推理智能调度与训推潮汐混部。

> [!info]- 参考资料
> - **Kubernetes 装箱评分与优先级抢占**：[Kubernetes v1.36 · 资源装箱评分](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/)、[Kubernetes v1.36 · 调度框架](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)、[Kubernetes v1.36 · Pod 优先级与抢占](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)、[Kubernetes v1.36 · 原生拓扑放置的抢占条件](https://v1-36.docs.kubernetes.io/docs/concepts/workloads/workload-api/topology-aware-scheduling/)
> - **Kueue 队列策略、借用与公平抢占**：[Kueue v0.17 · ClusterQueue 队列策略](https://kueue.sigs.k8s.io/v0.17/docs/concepts/cluster_queue/#queueing-strategy)、[Kueue v0.17 · ClusterQueue 与 Cohort 借用](https://kueue.sigs.k8s.io/v0.17/docs/concepts/cluster_queue/#cohort)、[Kueue v0.17 · 作业抢占](https://kueue.sigs.k8s.io/v0.17/docs/concepts/preemption/)、[Kueue v0.17 · Fair Sharing 加权份额](https://kueue.sigs.k8s.io/v0.17/docs/concepts/preemption/#fair-sharing)
> - **PyTorch 检查点与故障恢复注意项**：[PyTorch 2.9 · 故障、成员变化与检查点](https://docs.pytorch.org/docs/2.9/elastic/run.html#important-notices)
