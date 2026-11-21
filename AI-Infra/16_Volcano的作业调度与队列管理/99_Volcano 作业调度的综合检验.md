# Volcano 作业调度的综合检验

下面给出 5 组可脱离实机、完全依靠纸笔推演的综合检验题，覆盖入队门槛、份额装箱、同队列抢占与跨队列回收、故障重试与检查点、Kueue 准入对照，以及训推混部的潮汐算力回收。

除题目另有调整外，全篇沿用统一版本栈（Volcano `v1.16.0`、Kueue `v0.17`、Kubernetes `v1.36`、PyTorch `2.9`）与主线规格 `train-12`：

| 参数维度 | 基准规格说明 |
| --- | --- |
| 作业形态（`train-12`） | 3 只固定 worker Pod，硬性反亲和限定每节点最多 1 只；每只启动 4 个训练进程 |
| 单只 worker 请求 | `4` 张 H100 80 GB 整卡、`16` CPU、`96 GiB` 内存、`1` 个支持 RDMA 的 SR-IOV VF |
| 标准节点容量 | 同型单网络域 8 卡节点，每台 Allocatable 为 `8` GPU、`32` CPU、`192 GiB` 内存、`2` VF |

建议先合上参考推理，逐题写下算式与排查链条，再展开折叠块对照核验。

---

## 材料一：入队账本、角色门槛与节点余量

API 中已存在 `ai-demo/train-12` `VolcanoJob`，`spec.queue=train`，唯一 `Task` 名为 `worker`，`replicas` 与两级 `minAvailable` 均为 `3`。由该 Job UID 拥有的 `PodGroup` 处于 `Pending`，集群中尚无 worker Pod。

调度器配置 `enqueue, allocate`，启用 `capacity` 与 `gang`。队列 `train` 处于开放状态，GPU 上限（`capability`）为 `16`，当前已分配 `3` GPU，另有 `2` GPU 在途（`inqueue`），无弹性校正与专属保留，其他维度额度充足。节点实时余量如下：

| 节点 | GPU 余量 | CPU 余量 | 内存余量 | RDMA VF 余量 |
| --- | ---: | ---: | ---: | ---: |
| `node-1` | 7 | 24 | 160 GiB | 1 |
| `node-2` | 4 | 16 | 96 GiB | 1 |
| `node-3` | 5 | 24 | 128 GiB | **0** |

**思考与复算：**

1. 分别指出谁生成 `PodGroup`、谁将它推出 `Pending`、谁随后创建 3 只 worker Pod；写出应核对的组字段和当前未放行的直接原因。
2. 若另一作业撤回，调度器不再计入那 `2` GPU 在途需求（节点余量不变），复算入队与成组放置结果；如果仅让 `node-3` 归还 1 个 VF，结果如何变化？
3. 保持 VF 尚未归还的初始库存。若新提交作业只把 Job `minAvailable` 降为 `2` 而把 Task 门槛留在 `3`，Volcano `v1.16.0` 如何检查？若要明确表达“2 台即可开跑”，还须改动哪些规格与启动参数？2 个 Agent 会合后总进程数是多少？

> [!example]- 参考推理（材料一）
> 1. **组件分工与当前阻塞**：**Volcano Job Controller** 生成 `PodGroup`，写入 `spec.minMember=3`、`spec.minTaskMember.worker=3` 与 `spec.minResources`（`12` GPU、`48` CPU、`288 GiB`、`3` VF）。
>
>    由 **Volcano Scheduler** 执行 `enqueue`，当前账面总需求为：
>
> $$\text{账面总需求} = 12 + 3 + 2 = 17\text{ GPU} > 16\text{ GPU（队列上限）}$$
>
>    因超上限被拒，组留在 `Pending`；待放行进入 `Inqueue` 后，仍由 **Job Controller** 创建 3 只 worker Pod。
>
> 2. **撤回在途需求与归还 VF**：撤回 2 GPU 在途需求后，账面变为 `12 + 3 = 15 ≤ 16`，顺利入队并建出 3 只 Pod。虽然三台共余 `16` GPU，但 `node-3` 的 VF 余量为 `0`，仅 `node-1/2` 合格，Gang 门槛（3）未达标而回滚。
>
>    当 `node-3` 归还 1 个 VF 并归账后，三台全部合格，组进入绑定流程（随后还需等 kubelet 交付设备与 3 个 Agent 会合）。
>
> 3. **降级启动与门槛对齐**：只改 Job `minAvailable=2` 时，`v1.16.0` 因整组门槛（2）小于角色门槛之和（3）而跳过逐角色校验，最低需求按 2 副本（8 GPU）计，2 个落点即可提交绑定。
>
>    要规范表达弹性需求，应将 **Job 与 Task 的 `minAvailable` 均设为 `2`**（保留 `replicas: 3`），并将启动命令改为弹性 `--nnodes=2:3`；2 个 Agent 会合时启动 `2 × 4 = 8` 进程（仍用固定 `--nnodes=3` 则会空占 8 卡干等）。

---

## 材料二：队列份额、排序、装箱与可回收落点

以下三小题快照彼此独立，均基于 3 节点 24 GPU 池（每台 `8` GPU、`32` CPU），未提及维度均满足要求。

1. **专属保留与有效上限**：`train` 与 `batch` 的 GPU 上限分别为 `16` 与 `24`，应得份额分别为 `12` 与 `11`。在空池状态下为 `train` 设置专属保留 `guarantee.resource = 2 GPU`，求 `batch` 的 GPU 有效上限。
2. **DRF 排序**：同队列、同优先级的作业 A（`2 GPU + 24 CPU`）与 B（`4 GPU + 12 CPU`）前置回调打平、其余维度占比更低。求主导份额并定顺序；若 `priority` 在前且调高 A 的优先级，顺序怎么变？
3. **节点装箱**：将 10 只单卡 Pod 按 `4 / 3 / 3` 或 `8 / 2 / 0` 占用放入三节点，分别留给 `train-12` 几个 4 卡落点？若等待的是 1 只 8 卡整机 Pod，落点数又各是多少？
4. **跨队列回收与落点**：专属保留为 `0`，`batch`（`deserved=11`）在三节点占 `4 / 4 / 7` GPU；`train` 用量 `0`，`train-12` 已入队待放。`node-2/3` 各有 1 只 `batch` 合格 4 卡候选（限驱 1 只）。该用何种 action？复算两方案余量与成组结果，并说明 `reclaimable: false` 的影响。

> [!example]- 参考推理（材料二）
> 1. **有效上限**：扣除 `train` 的 2 GPU 专属保留后再按自身上限封顶：
>
> $$\text{有效上限}_{\text{batch}} = \min(24,\; 24 - 2) = 22\text{ GPU}$$
>
> 2. **DRF 排序**：全池共 `24` GPU、`96` CPU，两作业的主导份额分别为：
>
> $$\text{DRF}_A = \max\left(\frac{2}{24},\; \frac{24}{96}\right) = \frac{1}{4}, \qquad \text{DRF}_B = \max\left(\frac{4}{24},\; \frac{12}{96}\right) = \frac{1}{6}$$
>
>    因 `1/6 < 1/4`，默认 DRF 先考虑 **B**；若 `priority` 在前且提高 A 的优先级，则转为先考虑 **A**。
>
> 3. **装箱落点**：`4 / 3 / 3` 对应余量 `4 / 5 / 5`（余 14），留给 `train-12` **3 个** 4 卡落点，留给 8 卡 Pod **0 个**落点；`8 / 2 / 0` 对应余量 `0 / 6 / 8`（同余 14），留给 `train-12` **2 个**落点，却留给 8 卡 Pod **1 个**整机落点。
>
> 4. **跨队列回收**：跨队列应启用 **`reclaim`**；无论驱逐哪只 4 卡候选，`batch` 用量均由 `15` 降回 `11`。
>
>    但按节点复算：释出 `node-2` 后余量为 **`4 / 8 / 1`**，因反亲和仅得 2 个落点（**失败**）；释出 `node-3` 后余量为 **`4 / 4 / 5`**，凑齐 3 个落点（**成功**）。若设 `batch.spec.reclaimable: false`，回收筛选会直接排除 `batch` 的所有成员。

---

## 材料三：重试层级、检查点与队列对账

下表列出三次相互独立的运行记录：

| 记录编号 | 观测到的对象与日志信息 |
| --- | --- |
| **记录甲** | Pod UID 不变，容器 `restartCount=0`，`torchrun` Agent 日志显示重建了本轮 WorkerGroup。 |
| **记录乙** | 在允许容器重启的模板中，Pod UID 不变，`restartCount` 从 `0` 增为 `1`，`lastState.terminated.exitCode=1`。 |
| **记录丙** | Job 命中 `PodFailed → RestartJob`，`.status.retryCount` 从 `0` 增为 `1`，三只 worker 沿用原名但 UID 全部改变。 |

**思考与复算：**

1. 分别指出甲、乙、丙确认了哪一层的重试动作，以及下一步应重点核查的证据。
2. 在记录丙中，失败前训练已共同完成第 `9,420` 步；存储上第 `9,000` 步检查点完整可读，第 `9,400` 步仅存部分分片。计算需重做的全局步数。若失败前 `train` 队列共分配 `15` GPU，3 只旧 Pod 的 `12` GPU 全部释出，同时同队列另一新 Pod 获批 `4` GPU，队列账面应显示多少？怎样确认训练已真正恢复？

> [!example]- 参考推理（材料三）
> 1. **三层重试归因**：
>    - **记录甲**：**应用启动器层（`torchrun`）**重启了训练进程组。下一步核对本轮 `rdzv-id`、会合后的 `WORLD_SIZE` 以及重新加载的检查点路径。
>    - **记录乙**：**节点容器层（kubelet）**在同一 Pod 内重启了容器。下一步结合 kubelet 事件、退出码 `1` 对应日志与 `restartPolicy` 查容器退出根因。
>    - **记录丙**：**集群作业层（Volcano Job Controller）**执行了整组重建。下一步结合控制器事件与新旧 UID 确认旧成员全退，核对新 Pod 的 `spec.nodeName`、设备交付与新进程组会合。
> 2. **重做步数与净对账**：第 `9,400` 步残缺不可加载，必须回退到最近完整提交的第 `9,000` 步：
>
> $$\text{重做步数} = 9{,}420 - 9{,}000 = 420\text{ 步}$$
>
> 队列释放与并发新分配叠加后的最新账面为：
>
> $$\text{最新队列 } \texttt{allocated} = 15 - 12 + 4 = 7\text{ GPU}$$
>
> 表面净降 8 GPU 与旧成员完整释放 12 GPU 同时成立。确认恢复成功不能止步于 Pod `Running`，必须核对新一轮实际加载的存档版本、日志报告的恢复起始步数以及后续优化步持续推进。

---

## 材料四：Kueue 准入检查、逐 Pod 绑定与 Trainer 追踪

将同一份三节点训练需求迁移至 Kueue `v0.17` 管理的 Kubernetes `Job`，由默认调度器逐 Pod 绑定。物理节点仍采用**材料一的初始库存**（`node-3` 的 VF 余量为 `0`），且未给 `ResourceFlavor` 配置拓扑感知准入。

| 观测时点 | 集群对象状态 |
| --- | --- |
| **时点甲** | `Workload` 显示 `QuotaReserved=True`、`Admitted=False`，一项 `.status.admissionChecks` 为 `Pending`，`Job` 保持挂起。 |
| **时点乙** | 检查通过且 `Admitted=True`，`Job` 解挂并建出 3 只 Pod；其中 2 只已绑定节点，第 3 只的 `spec.nodeName` 仍为空。 |

**思考与复算：**

1. 分别为时点甲和时点乙写出当前卡点、负责组件与下一条要核查的证据。
2. 若改为采用 JobSet 的 **Kubeflow Trainer v2** Runtime，并由 Kueue 的 `TrainJob` 集成纳管，应从哪个顶层对象检查准入，沿哪一条所有权链一路追到具体 Pod 与绑定调度器？

> [!example]- 参考推理（材料四）
> 1. **两时点诊断**：
>    - **时点甲**：额度已锁定，卡在**外部准入检查（`AdmissionCheck`）**。负责推进的是该检查对应的外部控制器与 Kueue Admission 逻辑；下一步查阅 `.status.admissionChecks` 的名称与消息，待通过后确认 Kueue 解挂 `Job`。
>    - **时点乙**：已通过 Kueue 准入，卡在**底层默认调度器的逐 Pod 节点绑定**。下一步查默认 `kube-scheduler` 对第 3 只 Pod UID 上报的 `FailedScheduling` 事件，结合硬反亲和与节点库存复算——`node-1/2` 各已落 1 只，`node-3` 的 RDMA VF 余量为 `0`，导致第 3 只无法绑定。
> 2. **Trainer 级联追踪链**：从顶层 **`TrainJob`** 起查队列标签、`spec.suspend` 与关联 `Workload`，并核对 `spec.runtimeRef` 指向的 Runtime。
>
>    随后沿 `ownerReferences` 依次追踪 **`TrainJob → JobSet → 子 Kubernetes Job → Pod`**，最后读 Pod 的 `spec.schedulerName` 确认实际绑定者。

---

## 材料五：训推混部多队列潮汐回收推演

某生产集群由 4 台 8 卡节点（`node-1`–`node-4`，共 **32 GPU**）组成，同时服务在线大模型推理与离线批处理训练。Volcano `capacity` 双队列配置如下：

| 队列名 | `priority` | `reclaimable` | `guarantee` | `deserved` | `capability` |
| --- | ---: | --- | ---: | ---: | ---: |
| `infer-online` | 100 | `false` | `8` GPU | `24` GPU | `32` GPU |
| `train-offline` | 10 | `true` | `0` GPU | `8` GPU | `24` GPU |

夜间低谷期，`infer-online` 仅运行 2 个 4 卡推理副本（全部位于 `node-1`，共 `8` GPU）；`train-offline` 在 `node-2/3/4` 上分别占用 `8 / 8 / 4` GPU（共 `20` GPU）。

**思考与复算：**

1. 计算夜间 `train-offline` 的 GPU 有效上限，以及当前超出其应得份额的借用量。
2. 早高峰流量激增，HPA 将 `infer-online` 新增 3 个 4 卡副本（需增量 `12` GPU，总量达到 `20` GPU）。`reclaim` 能否合法触发？需向 `train-offline` 至少回收多少张卡？回收后 `train-offline` 是否触碰其 `deserved` 底薪？
3. 假若晚间某份离线训练任务被打上了极高的作业优先级，它能否反向跨队列回收 `infer-online` 正在运行的 8 卡推理副本？为什么？
4. 若将该集群的混部治理切换为 **Kueue `v0.17`（`v1beta2`）** 同一 `Cohort` 下的两个 `ClusterQueue`，分别用哪几个字段对应 Volcano 里的 `deserved`、`guarantee` 与单向回收规则？

> [!example]- 参考推理（材料五）
> 1. **夜间上限与借用量**：扣除 `infer-online` 的 `8` GPU 专属保留后，`train-offline` 有效上限与当前借用量为：
>
> $$\text{有效上限}_{\text{train}} = \min(24,\; 32 - 8) = 24\text{ GPU}, \qquad \text{借用量} = 20 - 8 = 12\text{ GPU}$$
>
> 2. **早高峰跨队列回收**：此时全池仅空余 `32 - 8 - 20 = 4` GPU（位于 `node-4`），推理扩容需 `12` GPU，物理缺口为 `8` GPU。
>
>    因 `infer-online` 目标 `20 ≤ deserved (24)`，且 `train-offline` 用量 `20 > deserved (8)` 并开启 `reclaimable: true`，`reclaim` 合法触发。
>
>    调度器至少回收 **`8` GPU**（合 `node-4` 空余 4 卡凑齐 3 个 4 卡槽位）；回收后 `train-offline` 仍剩 `12` GPU，**高于 `8` GPU 底薪**。
>
> 3. **单向防误伤保护**：**绝不能**。`infer-online` 已设 `reclaimable: false` 阻断跨队列回收，且这 8 卡属于其 `guarantee` 专属保留、也未超出 `deserved`（24）；作业优先级仅作用于同队列 `preempt`，无法突破跨队列硬边界。
>
> 4. **Kueue `v1beta2` 字段映射**：`deserved` 对应 `nominalQuota`（`24` vs `8`）；推理保留 8 卡不外借通过 `lendingLimit: 16`（`24 - 16 = 8`）实现。
>
>    单向回收由 `infer-online-cq` 配置 `reclaimWithinCohort: LowerPriority` 与 `train-offline-cq` 配置 `reclaimWithinCohort: Never` 共同落实。

---

## 全章知识串联小结

把这一章的所有机制压缩成一条主线，就是**从业务意图到底层算力的四级漏斗**：

1. **声明层（`VolcanoJob` / `Deployment` / `TrainJob`）**：把训练的同步人数或推理的副本弹性，翻译成标准化 Pod 模板、成组门槛与重试策略。
2. **账本层（Volcano `Queue` / Kueue `ClusterQueue`）**：通过上限、应得份额、专属保留与共享借还规则，隔离多租户并编排训推潮汐算力。
3. **落点层（`allocate` + `gang` + `binpack` / `preempt` / `reclaim`）**：把总量额度落实到具体物理节点的四维余量、反亲和与拓扑槽位，确保“要么凑齐成组落座，要么不白杀不死锁”。
4. **恢复层（`torchrun` / kubelet / `RestartJob` + Checkpoint）**：面对成员崩溃或潮汐回收，沿 UID 与检查点完成最小损失的重建与账面归零。

接下来在 [第 17 章 把设备、网络与作业状态联系起来](../17_把设备、网络与作业状态联系起来/00_内容大纲.md) 中，我们会把本章的对象状态与事件流正式接入 DCGM、网络与业务指标时间线，让“作业卡在队列、卡在启动，还是卡在硬件与网络”彻底一目了然。

> [!info]- 参考资料
> - **Volcano 队列、成组放置与回收动作**：[Volcano v1.16.0 Actions](https://volcano.sh/docs/v1.16.0/scheduler/actions/)、[Queue 概念](https://volcano.sh/docs/v1.16.0/concepts/queue/)、[PodGroup 概念](https://volcano.sh/docs/v1.16.0/concepts/podgroup/)、[Capacity 插件配置指南](https://volcano.sh/docs/v1.16.0/userguide/user_guide_how_to_use_capacity_plugin/)、[Volcano v1.16.0 Job Policy 指南](https://volcano.sh/docs/v1.16.0/userguide/user_guide_how_to_use_job_policy/)
> - **Kueue 队列对照与 PyTorch 弹性恢复**：[Kueue v0.17 · v1beta2 API 参考](https://kueue.sigs.k8s.io/v0.17/docs/reference/kueue.v1beta2/)、[Kueue v0.17 · 拓扑感知调度（TAS）](https://kueue.sigs.k8s.io/v0.17/docs/concepts/topology_aware_scheduling/)、[PyTorch 2.9 · 固定与弹性启动](https://docs.pytorch.org/docs/2.9/elastic/run.html)
