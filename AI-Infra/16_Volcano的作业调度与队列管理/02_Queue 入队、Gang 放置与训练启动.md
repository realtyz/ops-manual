
`train-12` 固定要三只 worker Pod，分别放在 `h1`、`h2`、`h3`。每台节点的 Allocatable（可供 Pod 请求的容量）为 8 GPU、32 CPU、192 GiB 主机内存、2 个支持 RDMA 的 VF。每个 worker 请求 4 张 H100 80 GB 整卡、16 CPU、96 GiB、1 VF。三台节点用同一可互通的网络域，这比第 15 章的多网络域例子少一层约束。本篇按 Volcano v1.16.0 的 `enqueue, allocate` 教学配置推演，启用 `capacity`、`predicates` 和 `gang`。以下快照中的余量都已扣除调度器计入的请求，除特别说明外，其余资源和网络条件满足。 [Volcano v1.16.0 Actions](https://volcano.sh/docs/v1.16.0/scheduler/actions/)

## Queue 有效上限与最低入队请求

`train` 是一组 PodGroup 共用的资源账本。它开放接纳作业，GPU 维度的使用上限（`capability`）为 16，应得份额（`deserved`）为 12。应得份额供共享竞争时使用，空闲资源可以让队列用到上限。下列配置只展示本例需要的 GPU 维度，其他维度的有效容量充足，且无任何队列的专属保留。 [Volcano v1.16.0 Queue](https://volcano.sh/docs/v1.16.0/concepts/queue/) · [Capacity 配置指南](https://volcano.sh/docs/v1.16.0/userguide/user_guide_how_to_use_capacity_plugin/)

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: train
spec:
  capability:
    nvidia.com/gpu: "16"
  deserved:
    nvidia.com/gpu: "12"
```

Job Controller 先为 Job 建立 PodGroup，写入 `minMember=3` 和 `minResources` 中 12 GPU 等最低请求。在本例的平级队列中，`enqueue` 调用 `capacity`，检查队列开放状态和各资源维度的有效上限。典型非空最低请求的账式为 `新组最低请求 + 已分配 + 在途 − 弹性校正`。在途表示已经放行且还需计入的资源，插件在统计时扣除同一组已分配的部分。弹性校正来自已有作业中超出最低规模的可弹性部分。本例两项均为 0，GPU 有效上限为 16。若检查通过，PodGroup 进入 `Inqueue`。Job Controller 看到组阶段已有值且已离开 `Pending`，才继续创建成员。 [v1.16.0 capacity 入队检查](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/plugins/capacity/capacity.go#L1621-L1667) · [v1.16.0 enqueue](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/actions/enqueue/enqueue.go) · [v1.16.0 Pod 同步条件](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/controllers/job/job_controller_actions.go#L400-L414)

## 节点筛选、暂选放置与 Gang 提交

NodeSelector 用标签选出 H100 80 GB 节点。Pod 的硬性反亲和按主机名划定范围，要求同一命名空间内带 `infra.example.com/workload: train-12` 标签的 Pod 分居不同节点。以下是 `spec.tasks[0].template.spec` 中的新增字段，标签应写在同一模板的 metadata。每节点最多一只本作业的 worker，所以一台余 8 卡的节点仍只贡献一个落点。 [Kubernetes v1.36 节点选择与 Pod 反亲和](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)

```yaml
nodeSelector:
  infra.example.com/gpu-model: h100-80gb
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            infra.example.com/workload: train-12
        topologyKey: kubernetes.io/hostname
```

在三个连续时点，分别看队列还有多少容量、节点能提供几个完整落点。

| 快照 | 已有 GPU 请求 | `train` 账面（GPU） | `h1 / h2 / h3` GPU 余量 | 可放 worker |
| --- | --- | --- | --- | ---: |
| A，提交作业 | `train` 在 `h1/h2` 各用 4 | `8 + 12 = 20 > 16`，继续等入队 | `4 / 4 / 8` | 3 |
| B，入队后 | `h1` 旧成员退出并归账，`h2` 的 4 卡保留。新成员首次参与放置前，`batch` 已在 `h3` 占 6 卡 | `4 + 12 = 16`，通过入队 | `8 / 4 / 2` | 2 |
| C，节点资源释放后 | `h3` 上 `batch` 的一个 2 卡 Pod 退出并归账 | 新组的 12 卡放入后合计 16 | `8 / 4 / 4` | 3 |

![快照 A 中 train 用量 8 加最低需求 12 超过上限 16，三个节点在容量上各有一个落点。快照 B 中 4 加 12 等于 16，作业已经入队，h3 仅余 2 张卡。](images/fig-16-02-01-queue-and-gang.png)

图 16-02-01　队列与节点的两道检查。快照 A 仍在等入队，图中的三个落点是假设已有待放置成员时的节点容量。快照 B 已可创建成员，但只有两个落点。图中的「可分配」指扣除已有请求后的调度余量。

对快照 B，`allocate` 读取三个待放 Pod，检查队列是否允许增加请求，在候选节点上检查资源向量、标签和反亲和，然后给合格节点打分。它可以暂选 `h1` 和 `h2`，并将暂选结果计入下一只 Pod 的判断。第三只被同组反亲和排除在前两台之外，`h3` 的 2 GPU 也小于 4。Gang（成组调度）插件检查整组和适用的角色门槛。本例两者都是 3，且没有已绑定的成员或即将释放的候选资源，因此本轮无法提交这两个暂选绑定。即使总余量 `8 + 4 + 2 = 14` 超过需求 12，结果也一样。 [v1.16.0 allocate 提交逻辑](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/actions/allocate/allocate.go) · [v1.16.0 Gang 的组及角色检查](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/plugins/gang/gang.go#L175-L196)

快照 C 可各选一台节点，整组和 `worker` 角色的门槛同时满足，调度器提交放置并为各 Pod 发起绑定。此后各节点分别启动容器，节点侧设备准备、镜像加载和应用启动有各自的状态和耗时。 [v1.16.0 调度事务](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/framework/statement.go) · [Kubernetes v1.36 Pod 生命周期](https://v1-36.docs.kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

## Pod 绑定、会合与进程组就绪

每个 worker Pod 内的 `torchrun` Agent 使用相同的会合后端、会合标识和可达端点。**会合**（Rendezvous）让本轮的三个 Agent 确认成员和分工，`--nnodes=3 --nproc-per-node=4` 表示三台节点、每节点四个进程。进程总数为 `WORLD_SIZE=3 × 4=12`，每个进程在本轮有一个全局编号 Rank，并按本地编号选用容器可见的 GPU。应用建立进程组后还需验证首次集合通信能够完成。DNS 可用时，可以用 Volcano 的可选 `svc` 作业插件协助成员发现，由应用配置选择其中实际提供会合服务的端点。 [PyTorch 2.9 torchrun](https://docs.pytorch.org/docs/2.9/elastic/run.html) · [Volcano v1.16.0 SVC 插件](https://volcano.sh/docs/v1.16.0/userguide/user_guide_how_to_use_svc_plugin/)

下面是三只 Pod 启动器使用同一组配置的示意命令。`rdzv.training.example` 代表本轮可达的会合服务主机，部署时替换为真实主机和端口。`train.py` 为镜像中的训练入口。

```bash
torchrun --nnodes=3 --nproc-per-node=4 --rdzv-backend=c10d \
  --rdzv-id=train-12-run1 --rdzv-endpoint=rdzv.training.example:29400 train.py
```

若只想在快照 B 的两台节点启动，需要区分三种条件。

| 新提交的规格 | 可提交的放置与训练结果 |
| --- | --- |
| 只把 Job 的 `minAvailable` 改成 2，Task 仍为 3 | v1.16.0 发现整组门槛 2 小于角色门槛之和 3，跳过逐角色检查。最低 GPU 请求按两个副本计为 8，两只可先绑定，固定三节点应用继续等。 |
| 同时把 Job 与 Task 的 `minAvailable` 改为 2，`replicas` 仍为 3 | 两只 Pod 能先绑定，最低 GPU 请求也降至 8。仍用 `--nnodes=3` 时，两个 Agent 等第三台。 |
| 两级门槛均为 2，且训练应用支持 `--nnodes=2:3` | 若本轮只有两个 Agent 完成会合，`WORLD_SIZE=8`。第三个在下一轮加入时，应用需要按新规模恢复和重新分工。 |

在本例的单角色规格中，前两行都可能先交付两只 Pod。将两级门槛都写成 2，可以明确表达最低规模。留空 Task 的门槛时，无分区策略的 v1.16.0 webhook 会按 `replicas` 补值。多角色作业若把整组门槛降到各角色之和以下，就可能让部分角色没有最低副本保障。选择门槛要回到应用能用哪些成员推进。 [v1.16.0 Task 默认化](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/webhooks/admission/jobs/mutate/mutate_job.go#L180-L211) · [v1.16.0 最低资源计算](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/controllers/job/job_controller_actions.go#L895-L928) · [v1.16.0 角色检查分支](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/api/job_info.go#L1056-L1128) · [PyTorch 2.9 会合与成员变化](https://docs.pytorch.org/docs/2.9/elastic/rendezvous.html)

## 入队、绑定和应用等待的证据

下面的只读命令用于对应作业、组、队列和成员。命令面向采用上述命名且已安装 Volcano 的集群。事件按 `regarding.uid`、报告组件和时间筛选，再解读 `reason` 与 `note`。 [Kubernetes v1.36 Event API](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/events/event-v1/)

```bash
kubectl get jobs.batch.volcano.sh train-12 -n ai-demo -o yaml
kubectl get podgroups.scheduling.volcano.sh -n ai-demo -o yaml
kubectl get queues.scheduling.volcano.sh train -o yaml
kubectl get pods -l infra.example.com/workload=train-12 -n ai-demo -o yaml
kubectl get events.events.k8s.io -n ai-demo -o yaml
```

| 当前证据 | 下一步 |
| --- | --- |
| 已有 Job，找不到由这个 UID 拥有的 PodGroup | 查 Job Controller 的同步错误与 Job 的 `.status.state.phase`。若 Job 本身写入失败，从提交的 API 返回查校验或 webhook。 |
| 组为 `Pending`，worker 尚未创建 | 查 `spec.minResources`、Queue `.status.state` 和 `.status.allocated`。账面似乎有余量时，进一步查 scheduler 会话中的 `inqueue`、`elastic` 和有效上限。 |
| 组已入队，worker 已生成，当前 Pod 的 `spec.nodeName` 为空 | 查 PodGroup `.status.conditions`、节点筛选事件和当前可落点数。若入队后仍无 worker，则先追控制器创建成员的结果。 |
| Pod 已绑定或容器已经启动 | 用 `PodScheduled`、节点事件和容器 `state` 区分 kubelet 准备与应用运行。若容器都已启动，核对 Agent 端点、会合标识、最终 `WORLD_SIZE` 和通信进度。 |

`.status.allocated` 可用于核对队列分配量。处于不同写入时点的 Queue 状态、PodGroup 条件和 Pod 事件需结合 UID 和时间读。Volcano v1.16.0 的 `capacity` 达到队列有效上限时可能给 PodGroup 记 `Unschedulable` 事件，具体资源和原因还要查看消息。 [v1.16.0 capacity 事件路径](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/plugins/capacity/capacity.go) · [Volcano v1.16.0 PodGroup 状态](https://volcano.sh/docs/v1.16.0/concepts/podgroup/)

快照 A 的解法需要改变队列账本，快照 B 的解法需要改变特定节点的余量。队列份额、作业顺序和节点评分分别作用在这两道检查的不同位置。
