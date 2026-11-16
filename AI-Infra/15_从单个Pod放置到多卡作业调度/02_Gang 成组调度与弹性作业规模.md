
固定规模的多卡训练要等成员齐备。以 `train-12` 为例，三个 worker Pod 分别落在不同节点，每个 Pod 的 Agent 启动四个训练进程，共需十二个进程。**Gang Scheduling**（成组调度）是在调度器里按一组 Pod 决定是否允许绑定的机制。它用控制器声明的成员归属和最少成员数，把应用需要的最小规模带入资源放置。 [Kubernetes v1.36 · Gang Scheduling](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)

## 部分放置怎样占住整组所需的容量

作业甲和乙各有三个待放置的 worker，集群暂时只有四台合格节点，每台只接纳一个。假设逐 Pod 调度按甲、乙、甲、乙的顺序完成绑定，结果是 `2 + 2`。两份作业各缺一台节点，同时合计占用四台。若一直没有新节点或资源释放，且控制器保留这些 Pod，两个固定规模的会合都无法完成。若成组调度先为甲确认三个落点，结果可以是 `3 + 0`，甲可以进入启动流程，乙等待三个落点重新凑齐。

## 成员试放置与绑定门槛

设成组门槛为 3。先看组内已经创建的成员数。即使有三台空机器，如果调度器当前只看到了两个新建成员，仍缺少一个要安排的 Pod。三个成员到齐后，调度器才有完整的三个请求可试放置。在 Kubernetes v1.36 的原生实现中，组调度周期在绑定前尝试放置组内候选 Pod，`Permit` 阶段根据成功试放置的数量检查 `minCount`。 [Kubernetes v1.36 · Gang Scheduling](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/) · [Kubernetes v1.36 · 调度框架](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)

1. 依据成员的资源和约束尝试选择节点。一台节点若已被本组的某个 worker 试占，就要把这份请求计入后续成员的检查。本例还要求不同 worker 使用不同节点，所以同一台机器不能再充当这个组的第二个落点。
2. 三个 worker 各有一个可行落点时，调度器允许这批成功放置的成员继续绑定。若只找到两个，组在本轮等待，试放置占用的调度容量会被释放，供其他调度尝试使用。
3. 通过成组检查后，各 Pod 独立完成绑定及节点侧启动。节点拓扑拒绝、镜像准备失败、设备分配出错或会合超时都可能使可用成员数落回门槛以下。

成组检查能消除主例中的“本轮只放下两个也先绑定”路径。如果组后来增加成员，调度器会在评估新成员时计入已放置的成员。默认组调度算法按次序试放置，在有不同规格或复杂亲和关系时，可能因试放顺序找不到原本存在的可行组合。绑定后仍要分别确认获分配设备的 Pod 数、启动成功的 Agent 数和本轮进程规模。 [Kubernetes v1.36 · PodGroup 调度周期与限制](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/) · [Kubernetes v1.36 · Pod 生命周期](https://v1-36.docs.kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

## 固定与弹性作业的最小规模

假设本轮只有两个合格节点，仍是每节点一个 Agent、每 Agent 四个训练进程。若应用支持从两台启动，并且作业控制器允许在仍缺第三个 Pod 时运行，才可以把成组门槛设为 2。这里的 `--nnodes` 指会合的节点数，乘以每节点进程数才是该次组队的 World Size。 [PyTorch 2.9 · 固定与弹性启动](https://docs.pytorch.org/docs/2.9/elastic/run.html)

| 启动规格 | 最小成组数 | 两节点快照的结果 |
| --- | ---: | --- |
| 固定 `--nnodes=3 --nproc-per-node=4` | 3 | 成组检查继续等待，三节点会合无法完成。 |
| 弹性 `--nnodes=2:3 --nproc-per-node=4` | 2 | 调度门槛可通过。若两个 Agent 在同一轮会合且第三个未加入该轮，启动后 `WORLD_SIZE=8`。 |
| 固定 `--nnodes=3 --nproc-per-node=4`，误将门槛改成 2 | 2 | 两个 Pod 可获准绑定，Agent 仍等第三个节点。 |

弹性会合达到最低成员数后可以短暂等待其他成员。若第三个节点在首次会合结束后才加入，torchrun 可停止当前进程组并按新规模重启，新的 World Size 为 12，原有成员的 Rank 也可能重排。应用需要按当前成员规模重新划分工作，并在重启后从可恢复检查点继续。降低调度门槛时，也要确认队列准入和控制器允许交付两节点规模。 [PyTorch 2.9 · Rendezvous 等待](https://docs.pytorch.org/docs/2.9/elastic/rendezvous.html) · [PyTorch 2.9 · 成员变化](https://docs.pytorch.org/docs/2.9/elastic/run.html#membership-changes)

## Kubernetes 原生 PodGroup 的启用条件

原生 **PodGroup** 是调度器识别同组 Pod 和成组策略的 API 对象，`schedulingPolicy.gang.minCount` 表示最小成员数。v1.36 的能力为 Alpha，使用 `scheduling.k8s.io/v1alpha2`，需要启用 `GenericWorkload` 和 `GangScheduling`。v1.37 的原生 Workload、PodGroup 和成组策略升为 Beta，使用 `scheduling.k8s.io/v1beta1`，由默认关闭的 `GenericWorkload` 管理。还需确认所用控制器或 Pod 模板确实把成员接入该机制。用 Volcano 时则查其自身的组对象与策略，第 16 章将从作业配置走完这条路径。 [Kubernetes v1.36 · Workload-Aware Scheduling](https://kubernetes.io/blog/2026/05/13/kubernetes-v1-36-advancing-workload-aware-scheduling/) · [Kubernetes v1.37 · Workload-Aware Scheduling](https://kubernetes.io/blog/2026/09/08/kubernetes-v1-37-advancing-workload-aware-scheduling/)

成组门槛决定需要几个落点。可用落点取决于每个 Pod 的多项资源能否在同一节点凑齐，以及所选节点能否满足组的网络拓扑要求。
