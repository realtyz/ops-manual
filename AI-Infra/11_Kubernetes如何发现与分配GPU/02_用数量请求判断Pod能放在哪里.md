# 用数量请求判断 Pod 能放在哪里

## 资源值先表达整数单位

传统设备插件把 GPU 接入 Kubernetes 的**扩展资源**机制。对本章整卡独占场景，`nvidia.com/gpu: 2` 表示要两个插件公布的设备单位；它不表示 2 GiB 显存、两倍某个固定算力，也不将显存合并成连续地址空间。第 14 章会改变资源单位的实现，这里固定一单位对应一张卡。

扩展资源按整数单位记账且不超卖。在 GPU 请求中，可只写 `limits`，省略的该资源 `requests` 会取相同值；也可显式写二者相同。只写 GPU `requests` 而不写对应 `limits`、写不相等的二者或写 `0.5` 张，均不符合本章传统 GPU 请求规则。规则见 [Schedule GPUs：Using device plugins](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#using-device-plugins)及 [1.34 资源管理文档的 Extended resources 小节](https://github.com/kubernetes/website/blob/release-1.34/content/en/docs/concepts/configuration/manage-resources-containers.md)；相等与必须有 limit 的约束也可对照 [v1.34.0 的 validateResourceRequirements](https://github.com/kubernetes/kubernetes/blob/v1.34.0/pkg/apis/core/validation/validation.go)。

以下是**自拟完整 Pod 教学配置**。假定 `lesson-dp` namespace 已存在；探针镜像与运行环境满足导读条件。主线原生 CDI 不要求 NVIDIA 包装器 handler，因此不加第 10 章 legacy 的 RuntimeClass：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-p
  namespace: lesson-dp
spec:
  restartPolicy: Never
  containers:
    - name: worker
      image: lesson/gpu-probe:cuda12.8.1
      command: ["gpu-probe", "--all-visible"]
      resources:
        requests:
          cpu: "2"
          memory: 4Gi
        limits:
          nvidia.com/gpu: 1
```

这次 GPU 有效 request 是 1。CPU 的 2、内存的 4Gi 分别参与自己的预算；`memory` 指主机内存，不是 GPU 显存。配置未给 GPU 型号约束，两台节点公布相同资源名时都可以进入数量筛选，显存大小差异不会由 `nvidia.com/gpu: 1` 自动表达出来。

## Allocatable 与已请求量分别是什么

先回到两台节点全健康的基线。下表是**同一时刻、自拟并已完成状态传播的教学快照**，只含普通并行容器，无 init container、重启式 sidecar、Pod overhead 或其他 GPU 消费者：

| 节点 | Capacity | Allocatable | 活跃 Pod 已请求 GPU | 本轮剩余数量预算 |
| --- | ---: | ---: | ---: | ---: |
| gpu-a | 4 | 4 | old/worker：2 | 4 − 2 = 2 |
| gpu-b | 2 | 2 | 无：0 | 2 − 0 = 2 |

这里的“已请求”是调度器账本中的承诺量，不能用 `nvidia-smi` 的 GPU 利用率代替。old/worker 此刻即使利用率为 0%，仍占两个请求单位。`kubectl describe node` 的 Allocated resources 区域汇总请求与限制，也不是实时设备繁忙程度。

在这个简化模型里，某节点满足新 Pod GPU 数量的必要条件是：

\[
q_{\mathrm{new}} \le A_{\mathrm{node}}-\sum q_{\mathrm{existing}}
\]

`A` 是该节点的 Allocatable；总和包含调度器已计入的现有请求。为了给出确定结论，本节另外给定 CPU、内存、污点、亲和等所有其他调度约束都满足，且无并发竞争。于是新 Pod 要 2 张时两节点都合格；要 3 张时两节点都不合格，尽管集群合计有 4 张剩余。

一个 Pod 的普通容器必须位于同一节点，不能从 gpu-a 取两张再从 gpu-b 取一张来凑 3。跨节点使用 GPU 需要多个 Pod/进程协作，第 15 章才讨论如何一起推进这些成员。

<!-- 配图 fig-11-02-01：置于本段后，对照资源上限不变、已请求量增长时剩余预算如何变化。 -->

## 一次请求不会改写节点的健康资源上限

现在在 gpu-a 上成功启动新 Pod `probe-p`，占用一张。它使用哪张卡是下一篇的问题。数量状态变成：

```text
# 自拟两类查询的相关字段摘要
kubectl get node gpu-a -o yaml
status:
  capacity:
    nvidia.com/gpu: "4"
  allocatable:
    nvidia.com/gpu: "4"

kubectl describe node gpu-a
Allocated resources:             # 下列是教学删节，只保留 GPU 行
  Resource          Requests  Limits
  nvidia.com/gpu    3         3
```

节点仍有四个健康设备单位，所以 Allocatable 仍为 4；总请求是 3；新请求可用预算是 1。把 Allocatable 直接改读成“3 张还空闲”，会重复或遗漏记账。请求释放后，总请求减少，健康设备上限无须随之增加。

这里还要区分两个状态层次：调度器已经将某 Pod 纳入节点账本，并不证明节点内设备分配与容器创建已成功。Pod 可能已绑定节点，却仍在准备设备；对应的请求也不能当作从未发生。

## 多容器、健康变化与结论条件

同一 Pod 有两个普通并行容器，分别请求 1 和 2，则本章模型的 Pod GPU 请求为 3，分配结果分别关联到各容器。两者不会因属于同一 Pod 就自动共享一张卡。init container 和可重启 sidecar 的有效请求有各自生命周期合并规则；本章不把简单求和外推到这些情况，也不用它们承担练习的隐藏条件。

健康变化还会使“健康未分配 ID 数”与上述标量预算产生差别。例如 U0/U1 已分配、U3 空闲但变坏，`A=3`，已有请求 2，预算与健康空闲都为 1。若变坏的是已分配 U1，仍为 `A=3`、已有请求 2，但健康未分配 ID 是 U2/U3 两个。调度器看到的标量预算仍只有 1，不能从 `3−2` 反推出每张卡的健康和所有权。

因此**数量筛选与节点内 ID 分配是互补的两道判断**：前者检查资源承诺，后者从当前设备集合和分配记录中选择可用 ID。状态传播、并发与节点内资源对齐还能使“曾经通过调度”与“后来能成功启动”不同。本章的确定计算均明确锁定快照；实际故障要继续读节点内材料。

[上一篇：从设备清单走到节点资源](01_从设备清单走到节点资源.md) · [下一篇：从 kubelet 分配走到容器交付](03_从kubelet分配走到容器交付.md)
