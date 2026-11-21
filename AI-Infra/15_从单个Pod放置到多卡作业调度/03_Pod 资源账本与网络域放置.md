# Pod 资源账本与网络域放置

监控大盘明明显示整个集群还空着 **30 张 GPU**，比三节点训练作业要求的 **12 张卡**多出一倍有余，可作业提交后却死活卡在 `Pending`。

这类“看大盘不缺卡，调度器却放不下”的困惑几乎每天都在 GPU 集群上演。原因在于调度器从不把全集群的空闲卡加起来当大水池，而是**逐节点核算多维资源账本，再把合格落点放进网络域拓扑里成组检验**。

## 多维资源账本：卡够了，不等于节点能接单

沿用三节点训练作业 `train-12` 的单 worker 规格：4 张 H100 80 GB 整卡、16 个 CPU 单位、96 GiB 主机内存与 1 个支持 RDMA 的 VF，并且三个 worker 必须分摊到不同物理节点。

下面是一个典型训练 Job 的 Pod 模板关键片段（省略镜像、启动命令及第 13 章讲过的附加网络注解）：

```yaml
metadata:
  labels:
    infra.example.com/training-run: train-12     # 标识同一训练作业的成员
spec:
  nodeSelector:
    infra.example.com/gpu-pool: h100-80gb        # 限定 H100 80 GB 独占整卡池
  affinity:
    podAntiAffinity:                             # 硬性反亲和：同作业每台机器至多放 1 个
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              infra.example.com/training-run: train-12
          topologyKey: kubernetes.io/hostname
  containers:
    - name: worker
      resources:
        requests:                                # 四维资源须在同一节点同时满足
          cpu: "16"
          memory: 96Gi                           # 指主机内存，而非 80 GB 单卡显存
          nvidia.com/gpu: 4                      # 扩展资源：请求与限制必须相等
          example.com/rdma-vf: 1                 # 设备插件上报的高速网卡 VF 资源名
        limits:
          cpu: "16"
          memory: 96Gi
          nvidia.com/gpu: 4
          example.com/rdma-vf: 1
```

读这份模板时，要分清四个经常混淆的细节：

- **节点标签池筛选**：`nodeSelector` 把候选范围限定在同型号独占整卡池中，避免混入异构卡或切分节点。如果单台机器混插不同型号且共用 `nvidia.com/gpu`，仅靠节点标签无法精确挑卡，此时需借助第 11 章介绍的 **DRA**（由驱动经 `ResourceSlice` 上报单卡属性，再按 `ResourceClaim` 精准匹配）。
- **主机内存与显存分离**：`memory: 96Gi` 申请的是宿主机 DRAM；H100 标称的 80 GB 是单卡显存规格，在传统扩展资源模型里靠节点标签或卡型池来表达。若集群启用了第 14 章的 MIG 或共享切分，资源名与每单位代表的显存/算力含义也须按资源池配置解读。
- **扩展资源的整数对齐**：`nvidia.com/gpu` 与 `example.com/rdma-vf` 均属于 Kubernetes 扩展资源，其 `requests` 与 `limits` 必须为相同整数。
- **硬性 Pod 反亲和**：按 `kubernetes.io/hostname` 禁止同一作业标签的多个 Pod 落在同一台机器，确保每节点至多承接 1 个 worker。

## 逐节点推导：30 张空闲卡为何只剩 3 个落点

调度器维护的每节点**资源余量**，等于该节点 `Allocatable` 扣除调度器已承诺（含正在绑定中的 Pod）的有效请求之和。哪怕在跑任务的实际 GPU 利用率只有 5%，只要 Pod 未退出，它占走的整卡依旧留在账本上。

对通过型号筛选的每个节点，单个节点最多能承接的完整 worker 数遵循**木桶短板公式**：四项余量分别除以单 Pod 请求后向下取整，再取最小值，最后叠加反亲和上限 1：

$$
N_{\text{slot}} = \min\!\left(1,\; \left\lfloor \frac{G_{\text{free}}}{4} \right\rfloor,\; \left\lfloor \frac{C_{\text{free}}}{16} \right\rfloor,\; \left\lfloor \frac{M_{\text{free}}}{96\text{ GiB}} \right\rfloor,\; \left\lfloor \frac{V_{\text{free}}}{1} \right\rfloor\right)
$$

把这套公式套用到当前双网络域 6 台节点的资源账本中，全池 30 张空闲卡的真实去向立刻一目了然：

| 节点 | 网络域 | GPU 规格 | GPU 余量 | CPU 余量 | 主机内存余量 | VF 余量 | 可放 worker 数 | 筛查结论 |
| --- | --- | --- | ---: | ---: | ---: | ---: | ---: | --- |
| `h-a1` | A | H100 80 GB | 8 | 32 | 192 GiB | 2 | **1** | 四项商均为 2，反亲和封顶为 1 |
| `h-a2` | A | H100 80 GB | 4 | 16 | 96 GiB | 1 | **1** | 四项余量恰好凑齐 1 个完整副本 |
| `h-a3` | A | H100 80 GB | 2 | 32 | 192 GiB | 2 | **0** | GPU 仅剩 2 张，$\lfloor 2/4 \rfloor = 0$ |
| `h-b1` | B | H100 80 GB | 4 | 32 | 192 GiB | 1 | **1** | CPU/内存够 2 个，GPU/VF 仅够 1 个 |
| `h-b2` | B | H100 80 GB | 4 | 12 | 80 GiB | 0 | **0** | 虽有 4 张卡，但 CPU、内存与 VF 全线短板 |
| `a-b3` | B | A100 40 GB | 8 | 32 | 192 GiB | 2 | **0** | 在 `nodeSelector` 阶段即被异构卡型排除 |

汇总全表就能看透“30 张空闲卡的假象”：

1. 全表空闲 GPU 合计 **30 张**；
2. 扣掉异构节点 `a-b3` 的 8 张 A100 后，同型号 H100 余量剩 **22 张**；
3. 再逐节点核对四维木桶与单机反亲和，真正能接单的合格落点只有 `h-a1`、`h-a2`、`h-b1` 共 **3 处**，恰好匹配整组所需（12 GPU、48 CPU、288 GiB 内存、3 VF）。

> [!tip] 工程直觉：多维短板往往藏在 CPU 与 VF 里
> 现场排障时看到 `h-b2` 这样“空着 4 张 H100 却放不进 4 卡作业”的节点，别只盯着 GPU 数量看——极可能是同机其他 CPU 密集型辅助任务吃光了主机内存、CPU，或者高速网卡 VF 配额早已分配完毕。

## 叠加网络域：为什么同域约束瞬间卡住 3 个落点

多节点同步训练每一步都要跨机执行集合通信。集群通常会依据真实机架与交换机拓扑划出**网络域**（如同一叶交换机或同组直连机架），给节点打上 `infra.example.com/fabric-domain: a` 或 `b` 标签，让强同步作业尽量待在同一个高速通信域内。

把前面算出的 3 个合格落点按 A、B 两个网络域重新归组，局势立刻翻转：

| 集群状态与拓扑约束 | A 域落点 | B 域落点 | 3 个 worker 的放置结果 |
| --- | ---: | ---: | --- |
| 原余量，**硬性同域** | 2（`h-a1`、`h-a2`） | 1（`h-b1`） | 最大单域容量 $\max(2, 1) = 2 < 3$，整组继续等待 |
| 原余量，**允许跨域** | 2 | 1 | 跨域合计 $2 + 1 = 3$，两台落 A 域、一台落 B 域 |
| `h-a3` 释放 2 张 H100，**硬性同域** | 3（`h-a1`、`h-a2`、`h-a3`） | 1 | A 域三台节点各承接 1 个，满足硬性同域 |

这里藏着一个极其关键的空间规律：**释放同样 2 张 H100，释放位置不同，效果天差地别**。`h-a3` 原本只差 2 张卡，补齐后 GPU 达到 4 张，立刻给 A 域新增第 3 个落点；如果这 2 张卡是在已经满配反亲和上限的 `h-a1` 上释放的，`h-a1` 依然只能放 1 个 worker，A 域还是凑不齐 3 台。

![左右两组快照都有 A、B 两个网络域。左图 A 域可放两个 worker，h-a3 只剩两张 GPU。右图 h-a3 的余量增至四张，A 域可放三个 worker。B 域两次都只有 h-b1 可放一个。](images/fig-15-03-01-network-domain-placement.png)

*图 15-03-01：固定 3 节点训练作业在不同网络域余量下的放置结果对照。图中“可分配”指扣除已承诺请求后的节点余量。*

## 同域硬约束、偏好回退与推理的“反向打散”

在工程中，网络域约束通常分为三档策略：

- **同域硬约束**：必须在一个网络域内找齐全部成员，否则整组等待，适用于对跨 Spine 带宽收敛比高度敏感的大规模同步训练。
- **同域偏好与逐级放宽**：优先在最小网络域内凑齐；若放不下，则按拓扑层级向外放宽直至跨域，前提是跨域 RDMA 路由可达且性能达标。
- **直接跨域放置**：不限制网络域，仅满足单节点资源与反亲和即可，吞吐更易受跨域拥塞波动影响。

在 `Kubernetes v1.36` 中，原生 **Topology-Aware Workload Scheduling**（Alpha，需开启 `TopologyAwareWorkloadScheduling` 特性门控）可配合原生 Gang 策略，按 PodGroup 声明的拓扑标签在候选域内校验是否凑够 `minCount`。

若使用 `Kueue v0.17`，则可通过 **TAS（Topology Aware Scheduling，拓扑感知调度）** 在准入层提前核算整组 PodSet 的物理域容量：

- `kueue.x-k8s.io/podset-required-topology`：强制整组落在单一目标拓扑域内；
- `kueue.x-k8s.io/podset-preferred-topology`：放不下时允许按配置逐级向上放宽层级。

> [!warning] 常见误区：准入层与调度层的约束口径错配
> 在上面的原余量快照里，如果某个准入估算只按节点卡数之和算落点，会把有 8 张空闲卡的 `h-a1` 误算成 2 个落点，加上 `h-a2` 以为 A 域够放 3 个；结果真正进入调度器执行硬性反亲和时，`h-a1` 只能收 1 个，作业还是会卡住。核对拓扑准入记录时，务必把反亲和等 Pod 约束一并纳入。

最后把目光转向**在线大模型推理**，你会发现它的拓扑方向正好与训练相反：

- **训练追求“拉近聚合”**：把同一作业的多个 worker 尽量收拢到同一个高速网络域，压缩跨机集合通信跳数。
- **推理追求“打散容灾”**：同一模型的多个在线推理副本几乎不做跨副本同步通信，反而害怕单台交换机或机架断电导致服务全灭。因此在线推理通常配置 Kubernetes 原生的 `topologySpreadConstraints`，故意把副本均匀打散到不同节点、机架甚至可用区，以换取高可用性。

> [!summary] 核心结论
> 集群级调度画出了“哪几台节点能放、落在哪个网络域”的宏观边界。但调度器只看节点级总量，看不见单台服务器内部两颗 CPU 的 NUMA 切分——Pod 绑定上 `h-a2` 之后，节点侧的 kubelet 还有一道更精细的硬件关卡。

> [!info]- 参考资料
> - **Kubernetes GPU 调度、资源账本与反亲和**：[Kubernetes v1.36 · GPU 调度](https://v1-36.docs.kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)、[Kubernetes v1.36 · 扩展资源管理](https://v1-36.docs.kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources)、[Kubernetes v1.36 · 节点选择与 Pod 反亲和](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)、[Kubernetes v1.36 · Node Allocatable](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable)、[Kubernetes v1.36.0 · NodeResourcesFit 源码](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go)、[Kubernetes v1.36.0 · 调度器缓存源码](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/backend/cache/cache.go)
> - **DRA、共享切分与 SR-IOV 资源池**：[Kubernetes v1.36 · DRA](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)、[NVIDIA GPU Operator v26.7.2 · GPU 共享](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html)、[NVIDIA GPU Operator v26.7.2 · MIG](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-mig.html)、[SR-IOV Network Device Plugin · 资源池配置](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin)
> - **拓扑感知调度与分布约束**：[Kubernetes v1.36 · Pod 拓扑分布约束](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)、[Kubernetes v1.36 · 拓扑感知 Workload 调度](https://v1-36.docs.kubernetes.io/docs/concepts/workloads/workload-api/topology-aware-scheduling/)、[Kubernetes Workload-Aware Scheduling 功能开关演进](https://kubernetes.io/blog/2026/09/08/kubernetes-v1-37-advancing-workload-aware-scheduling/)、[Kueue v0.17 · TAS 拓扑感知调度](https://kueue.sigs.k8s.io/v0.17/docs/concepts/topology_aware_scheduling/)
