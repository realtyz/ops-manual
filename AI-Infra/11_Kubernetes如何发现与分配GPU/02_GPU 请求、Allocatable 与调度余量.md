
在 Device Plugin 的整卡路径中，节点上报的 `Allocatable` 给出 GPU 资源的可分配总量。调度器为一个新 Pod 选择节点时，还要扣掉调度快照中已经计入的请求。下面用 Kubernetes v1.36 的默认调度器和普通单容器 Pod 展开这个计算。 [Kubernetes v1.36 扩展资源管理](https://v1-36.docs.kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources) · [NodeResourcesFit v1.36.0](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go)

## 单容器的 GPU 数量请求

这个教学 Pod 省略了型号条件，镜像地址是占位。它用扩展资源 `nvidia.com/gpu` 声明需要一个完整 GPU 单位。该资源数量应为整数，并且不可超额承诺。仅声明 limit 时，在没有其他默认机制设置 request 的情况下，Kubernetes 将它用作 request。两项都写时值必须相同。如果提交给校验的对象只有 GPU request、缺少对应 limit，或两者不相等，校验会拒绝。 [Kubernetes v1.36 GPU 请求规则](https://v1-36.docs.kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#using-device-plugins) · [v1.36.0 资源字段校验](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/apis/core/validation/validation.go)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe
  namespace: gpu-example
spec:
  containers:
    - name: workload
      image: registry.example/gpu-probe:v1
      resources:
        limits:
          nvidia.com/gpu: 1
```

## 单节点的资源不等式

把新 Pod 的 GPU 请求记作 `q`，把某节点上已计入的有效 GPU 请求总量记作 `R`，把该节点的 GPU `Allocatable` 记作 `T`，三者都以整卡资源单位计。接受新请求之后的总承诺为 `R + q`。节点的数量检查要求 `R + q ≤ T`，因此新增请求 `q` 必须满足 `q ≤ T − R`。正数请求在账面差额为零或负数时都无法通过。默认调度器的 `NodeResourcesFit` 在本例按这一关系检查标量资源。它用调度时的 Node 和 Pod 请求状态，可能包含调度器已暂记到该节点、正在完成绑定的 Pod。 [NodeResourcesFit v1.36.0 数量检查](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go) · [v1.36.0 调度器缓存](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/backend/cache/cache.go)

| `gpu-h` 的教学快照 | Capacity | `T` | `R` | `T − R` | 请求一张 | 请求两张 |
| --- | ---: | ---: | ---: | ---: | --- | --- |
| A、B 健康，均未分配 | 2 | 2 | 0 | 2 | 通过 | 通过 |
| A 已占用，B 健康且未分配 | 2 | 2 | 1 | 1 | 通过 | 不通过 |
| A 的请求仍计入，B 标为不健康 | 2 | 1 | 1 | 0 | 不通过 | 不通过 |

表中每行都假定健康变更和已有请求已进入调度器所用的快照。第二行即使 A 的运行利用率是 0%，A 的 Pod 请求仍占一个单位。第三行由 B 的健康变化降低了 `T`，A 的请求照常计入 `R`。用 `nvidia-smi` 看到的利用率衡量设备活动，用 Pod 请求和 Node 状态推导调度余量。

![gpu-h 的两份快照都保留 A、B。A 健康且已有 Pod 使用。B 从健康变为不健康后，Capacity 仍为 2，Allocatable 变为 1，扣除已有请求后的余量从 1 变为 0。](images/fig-11-02-01-gpu-scheduling-capacity.png)

图 11-02-01　B 不健康后仍计入 Capacity。左右两侧 A 的请求均占一个单位；两列差额分别对应表格的第二行和第三行。

## 调度事件与单节点约束

如果 `gpu-h` 和 `gpu-a` 各有一个余额，整个集群合计两个空位，但一个请求两张卡的 Pod 仍没有可行节点。调度器对每个候选节点分别比较 `T − R` 与请求量，因为这个 Pod 的容器要在同一台节点启动。数量充足的节点还须满足 CPU、内存、污点容忍及节点选择等其他约束。 [Kubernetes v1.36 Pod 调度概念](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/) · [NodeResourcesFit v1.36.0](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go)

对于还未放置的 Pod，可从 `PodScheduled=False` 和 `FailedScheduling` 事件定位这轮调度的过滤原因。事件中的 `Insufficient nvidia.com/gpu` 表明有节点在数量检查失败。若事件同时列出节点选择器不匹配，需要回到各节点分别复算请求余量、比对标签，才能解释每台节点为何被排除。对于请求一张卡的健康主例，两台节点的数量都可能足够，下一步是把硬件型号也写进放置条件。 [Kubernetes v1.36 Pod 条件](https://v1-36.docs.kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions) · [NodeResourcesFit 的失败原因](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go)
