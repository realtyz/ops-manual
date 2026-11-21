# 队列上限、GPU 碎片与同域放置

早高峰 `08:45`，主训练作业刚刚提交入库。打开集群监控总览，6 台服务器账面上足足空着 **34 张整卡**；到了 `08:50`，空卡总数甚至涨到了 **38 张**（仅 A 域 H100 就空着 26 张）。可一份只需要 12 张卡的训练作业，却纹丝不动地僵在等待状态。

这正是训推共存生产集群里最常引发业务方投诉的“幽灵排队”现象。只看全局空闲卡数，永远看不出真相——我们必须把排查拆成先后两关：**队列账本能不能收单**，以及**早高峰在线推理占位之后，A 域到底还剩几台完整落点**。

## 第一关：队列配额先把 Pod 拦在门外（08:45 → 08:50）

集群通过独立的平级队列隔离两类负载：在线推理池使用高优先级 `infer` 队列，训练池使用 `train` 队列。`train` 队列当前开放且无专属预留，其 GPU 有效上限为 16（CPU、内存与 VF 维度均充裕，无在途弹性扣减）。

Volcano `v1.16.0` 的 `capacity` 插件会在创建 Pod 前，用 `PodGroup` 的整组最低请求（12 GPU、48 CPU、288 GiB、3 VF）核对队列上限。本例的 GPU 入队判据非常直白：

$$
G_{\text{allocated}} + G_{\text{minReq}} \le G_{\text{cap}} = 16
$$

按这条公式顺着时间轴核对 `train` 队列的账本变化，前 5 分钟的卡顿立刻水落石出：

| 时刻 | `train` 队列现有占用 | 队列核算结果 | 作业与 `PodGroup` 状态 |
| --- | --- | --- | --- |
| **08:45** | 2 个旧训练 Pod（分占 `node-1` 与 `node-5`，各 4 GPU） | $8 + 12 = 20 > 16$（超限） | `PodGroup` 停在 `Pending`，**0 个 worker Pod 被创建** |
| **08:50** | `node-1` 旧训练 Pod 退出归账，仅剩 `node-5` 占 4 GPU | $4 + 12 = 16 \le 16$（通过） | `PodGroup` 转为 `Inqueue`，控制器建出 3 个未绑定 Pod |
| **09:00** | 节点成组放置满足，3 个 Pod 一次性绑定 | 收敛后队列用量 $4 + 12 = 16$ | `PodGroup` 转 `Running`，队列 GPU 配额恰好打满 |

在终端排查时，只需连续观察 `Queue` 与 `PodGroup` 状态，就能瞬间区分作业到底是“被队列拦住”还是“正在找节点”：

```text
# 08:45：队列超限，PodGroup 停在 Pending，集群里连 worker Pod 都没有
$ kubectl get podgroup -n ai-demo
NAME             STATUS    MINMEMBER   RUNNING   PENDING
train-12-pg      Pending   3           0         0       # 未过 enqueue，不创建 Pod

# 08:50：node-1 旧任务退场释放 4 卡，PodGroup 进入 Inqueue，建出 3 个待绑定 Pod
$ kubectl get pod -n ai-demo -l volcano.sh/job-name=train-12 -o wide
NAME                READY   STATUS    NODE
train-12-worker-0   0/1     Pending   <none>             # spec.nodeName 为空，转入成组放置
train-12-worker-1   0/1     Pending   <none>
train-12-worker-2   0/1     Pending   <none>
```

> [!tip] 工程直觉：用两眼分清“卡队列”和“卡节点”
> 看到 `PodGroup` 为 `Pending` 且旗下还没有任何 worker Pod，说明卡在 **Queue 准入**，此时去查节点空卡纯属白忙；一旦 `PodGroup` 变为 `Inqueue` 且 3 个 Pod 已经建出来却没有 `NODE`，排查重心就应立刻切到**逐节点多维余量与成组放置**。

## 第二关：在线推理占位、VF 耗尽与同域碎片（08:50）

`08:50` 队列放行后，为什么 3 个 worker Pod 还是原地等了整整 10 分钟？翻开这一刻全集群 6 台节点的**调度余量**（`Allocatable` 扣除已调度承诺请求，注意监控里空转的卡只要已被分配就不算余量），早高峰训推共存的真实博弈立刻浮现出来：

| 节点 | 网络域 | 整卡池规格 | GPU 余量 | CPU 余量 | 内存余量（GiB） | VF 余量 | 同机现有占用（为什么剩这些余量） | 本作业可落点数 |
| --- | --- | --- | ---: | ---: | ---: | ---: | --- | ---: |
| `node-1` | A | H100 80 GB | 8 | 32 | 192 | 2 | 旧训练刚归账，整机空闲 | **1** |
| `node-2` | A | H100 80 GB | 8 | 32 | 192 | 2 | 整机空闲 | **1** |
| `node-3` | A | H100 80 GB | 2 | 24 | 168 | 1 | 常驻 4 卡在线推理 `infer-llm`（占 1 VF）+ 2 卡早间离线批量推理（0 VF） | **0** |
| `node-4` | A | H100 80 GB | 8 | 24 | 160 | 0 | RoCE 链路巡检/压测 Pod（0 GPU，占满 2 VF） | **0** |
| `node-5` | B | H100 80 GB | 4 | 16 | 96 | 1 | `train` 队列另一 B 域 4 卡训练 Pod | **0** |
| `node-a1` | B | A100 40 GB | 8 | 32 | 192 | 2 | B 域常规/量化推理缓冲池 | **0** |

调度器不会拿“总空卡 38 张”做算术，而是按固定顺序对每台主机过一遍筛子：

**第一步，先卡死硬件规格与网络域硬标签。** 模板通过 `nodeSelector` 锁定 H100 80 GB 与 A 域。B 域的 A100 节点单卡显存只有 40 GB，哪怕空着 8 张卡也装不下单卡需要 80 GB 显存的训练进程；B 域的另一台 H100 节点虽然四维资源样样达标，但跨出已验收的 A 域，直接出局。

**第二步，逐台计算“木桶最短板”落点数。** 对每台通过初筛的 A 域主机，把四项资源余量分别除以单个 worker 的请求量向下取整，再叠加“每机至多放 1 个同组 worker”的硬性反亲和封顶：

$$
N_{\text{slot}}(i) = \min\!\left(1,\; \left\lfloor\frac{\text{GPU}_i}{4}\right\rfloor,\; \left\lfloor\frac{\text{CPU}_i}{16}\right\rfloor,\; \left\lfloor\frac{\text{Mem}_i}{96}\right\rfloor,\; \left\lfloor\frac{\text{VF}_i}{1}\right\rfloor\right)
$$

代入 A 域四台主机的真实余量，四个截然不同的工程约束同时现形：

- **两台满血空闲主机（`node-1`、`node-2`）**：四项余量原本足够塞下 2 个 worker，但主机反亲和把每台上限封死在 **1**，两台合计贡献 2 个落点。
- **被在线推理切碎的混部主机（`node-3`）**：早高峰常驻的 4 卡在线推理副本绝不能随意驱逐，再加上 1 个 2 卡离线批量推理任务，使得 GPU 余量缩到 2 张，$\lfloor 2 / 4 \rfloor = 0$——典型的**训推混部 GPU 碎片**。
- **满机空卡却无网可接的主机（`node-4`）**：8 张 H100 全空，但 RoCE 巡检 Pod 占满了全部 2 个 VF，$\lfloor 0 / 1 \rfloor = 0$。同一 Pod 的设备必须在同一主机完成本地交付，**绝不可能从这台借 4 张 GPU、再去隔壁借 1 个 VF**。

结果非常残酷：全集群空着 38 张卡、A 域 H100 空着 26 张卡，完整合格的 A 域落点却只有 **2 个**。Volcano `allocate` 虽能试选两台空闲主机，但过不了 `gang` 的 3 节点门槛，于是果断放弃部分绑定，避免 8 张 H100 陷入“占卡死等第 3 个成员”的僵局。

> [!tip] 训推双重视角：同一张 08:50 快照，在线推理扩容为何畅通无阻？
> 假如同一时刻早高峰流量突增，需要立刻为 `infer-llm` 扩出 1 个 4 卡在线推理副本，它完全不会被卡住：第一，它走独立的 `infer` 队列，不受 `train` 队列 16 卡上限牵制；
>
> 第二，推理副本彼此独立，不需要 Gang 等齐三台；第三，推理服务追求跨节点容灾而非锁定 A 域，直接落到 B 域的 `node-5`（余 4 GPU / 16 CPU / 96 GiB / 1 VF）即可预热接客。

## 精准释放破局与三种单变量对照（08:56 → 09:00）

转机出现在 `08:56`：混部主机上那个不占 VF 的 2 卡离线批量推理任务正常完工归账，净释放 `2 GPU / 4 CPU / 8 GiB`，而同机常驻的 4 卡在线推理服务丝毫不受影响。

该节点的调度余量随即补齐为 `4 GPU / 28 CPU / 176 GiB / 1 VF`，GPU 短板消失，A 域一举凑出第 3 个完整落点。`09:00`，调度器一次性完成三台合格主机的成组绑定（见下图）。

![A 域四台节点在 08:50 与 08:56 的余量对比：h1 与 h2 两次均各有 8 GPU 和 2 VF 可供 1 个落点；h3 从 08:50 的 2 GPU 1 VF 缺 GPU 变为 08:56 的 4 GPU 1 VF 凑出第 3 个落点；h4 两次均有 8 GPU 和 0 VF 缺 VF。](images/fig-20-02-01-gang-placement-resource-fit.png)

*图 20-02-01：`08:50` 与 `08:56` 的 A 域节点余量与合格落点对照（图中 `h1`–`h4` 对应 `node-1`–`node-4`，CPU 与内存充足）。`08:56` `node-3` 归还 2 卡凑齐第 3 个落点，`node-4` 仍缺 VF；绑定于 `09:00` 落实。*

为了检验自己是否真正吃透“为什么同样释放资源，位置不同结果天差地别”，不妨回到 `08:50` 快照（此时 `train` 队列额度已过关），分别独立改动下面三项条件：

| 仅改变这一项条件 | 整组能否成组放置 | 背后的机制判据 |
| --- | --- | --- |
| **在 B 域节点（`node-5`）多释放 2 张 GPU** | **仍等待** | B 域不满足硬性同域选择，A 域合格落点依然只有 2 个。 |
| **节点 `node-4` 的巡检释放 1 个 VF**（CPU/内存仍够） | **立刻放行** | 全集群空闲 GPU 总数 1 张未增，但 `node-4` 的短板打通，A 域立刻凑齐第 3 个落点。 |
| **只把 Volcano `minAvailable` 降为 2** | **绑定 2 个后死锁** | 调度器放行了 2 个 Pod，但固定规模启动器仍死等第 3 个 Agent，8 张 H100 白白空转；除非应用同步支持弹性缩容重启。 |

> [!warning] 常见误区：调度绑定成功 ≠ 本地一定交付成功
> `09:00` 拿到三台节点的绑定结果，只是集群级账本结算完毕。如果节点 `node-3` 在绑定后的瞬间恰好丢失 VF 健康状态，或者附加网络 IPAM 地址池耗尽，Pod 仍会停在容器创建阶段——届时要顺着第 01 篇的 kubelet 设备管理器与 Multus/CNI 日志往下查。

本例三台主机的本地交付全部顺畅通过，`09:03` 顺利迈出第 1 个训练步。可稳态跑了不到半小时，监控曲线突然掉头向下——下一篇，我们直奔现场拆解那次诡异的训练步退化。

> [!info]- 参考资料
> - **Volcano 队列准入与成组放置**：[Volcano v1.16.0 Queue 概念](https://volcano.sh/docs/v1.16.0/concepts/queue/)、[Volcano v1.16.0 Enqueue 与 Allocate](https://volcano.sh/docs/v1.16.0/scheduler/actions/)、[Volcano v1.16.0 capacity 入队检查源码](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/plugins/capacity/capacity.go#L1621-L1667)、[Volcano v1.16.0 成员创建条件源码](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/controllers/job/job_controller_actions.go#L400-L414)、[Volcano v1.16.0 allocate 分配提交源码](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/actions/allocate/allocate.go)、[Volcano v1.16.0 gang 门槛源码](https://github.com/volcano-sh/volcano/blob/v1.16.0/pkg/scheduler/plugins/gang/gang.go#L175-L196)
> - **Kubernetes 节点选择、反亲和与设备健康**：[Kubernetes v1.36 Assign Pods to Nodes](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)、[Kubernetes v1.36 Device Plugin 不健康设备处理](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#device-plugin-and-unhealthy-devices)、[NVIDIA Device Plugin v0.20.1 共享与资源名说明](https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.20.1/README.md)
> - **PyTorch 固定与弹性规模语义**：[PyTorch 2.9 torchrun 弹性启动说明](https://docs.pytorch.org/docs/2.9/elastic/run.html)
