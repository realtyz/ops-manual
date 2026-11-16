
`gpu-h` 的两张整卡是 H100，`gpu-a` 的两张是 A100。两台节点都能上报 `nvidia.com/gpu: 2`，这个扩展资源的键值只有资源类别和数量。为了让需要 H100 的 Pod 选到合适节点，可以把型号作为节点特征发布，再在 Pod 中加一个节点选择器。 [Kubernetes v1.36 异构 GPU 集群调度](https://v1-36.docs.kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#manage-clusters-with-different-types-of-gpus)

## 节点特征的来源与含义

**Node Feature Discovery（NFD，节点特征发现）** 发现 CPU、PCI 等主机特征，并按配置将特征发布为 Node 标签。以 NVIDIA Device Plugin v0.19.2 官方 chart 的 NFD 配置为例，PCI 来源匹配指定设备类，标签名只取厂商 ID。匹配到 NVIDIA 设备可产生 `feature.node.kubernetes.io/pci-10de.present: "true"`，其中 `10de` 是 NVIDIA 的 PCI 厂商 ID。这个标签只告诉读者存在符合配置的 PCI 设备。 [NFD PCI 特征](https://nfd.sigs.k8s.io/usage/features.html#pci) · [NFD worker 配置](https://nfd.sigs.k8s.io/reference/worker-configuration-reference#sourcespci) · [NVIDIA v0.19.2 chart 配置](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.19.2/deployments/helm/nvidia-device-plugin/values.yaml)

**GPU Feature Discovery（GFD，GPU 特征发现）** 读取 NVIDIA 设备的型号、总显存等属性。GFD 启用时，v0.19.2 默认向 NFD 的 `NodeFeature` API 写入特征，由 NFD 将其投影到 Node 标签。是否部署 GFD 和是否用 `NodeFeature` 发布是两个配置项。以下摘自教学中的 `gpu-h`，型号字符串与容量采用示意值，匹配时应读取节点的实际标签值。 [GFD v0.19.2 发布方式](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.19.2/cmd/gpu-feature-discovery/main.go) · [GFD v0.19.2 整卡标签生成](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.19.2/internal/lm/resource.go)

```yaml
metadata:
  name: gpu-h
  labels:
    feature.node.kubernetes.io/pci-10de.present: "true"
    nvidia.com/gpu.product: NVIDIA-H100-80GB-HBM3
    nvidia.com/gpu.count: "2"
    nvidia.com/gpu.memory: "81920"
status:
  capacity:
    nvidia.com/gpu: "2"
  allocatable:
    nvidia.com/gpu: "2"
```

在本例的同型整卡配置里，`nvidia.com/gpu.count` 是 GFD 发现的设备数，`nvidia.com/gpu.memory` 是单卡总显存的 MiB 数。示意值 81920 MiB 除以 1024 等于 80 GiB。标签描述节点特征，kubelet 根据 Device Plugin 设备清单维护资源状态。于是，设备存在和型号标签可以保持可见，同时插件还未成功上报可分配资源。NFD 也允许通过 `NodeFeatureRule` 显式发布扩展资源，本例使用的是 NFD/GFD 标签与 Device Plugin 资源这组配置。 [GFD v0.19.2 标签生成](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.19.2/internal/lm/resource.go) · [NFD 扩展资源规则](https://nfd.sigs.k8s.io/usage/features.html#extended-resources)

## 型号与数量共同限定节点候选

将型号条件加入单卡 Pod。下面截取 `spec`，省略对象头，镜像地址仍为占位。`nodeSelector` 要求节点标签键值精确匹配。扩展资源请求由调度器用来检查能否新增一个单位。 [Kubernetes v1.36 nodeSelector](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) · [Kubernetes v1.36 GPU 请求规则](https://v1-36.docs.kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#using-device-plugins)

```yaml
spec:
  nodeSelector:
    nvidia.com/gpu.product: NVIDIA-H100-80GB-HBM3
  containers:
    - name: workload
      image: registry.example/gpu-probe:v1
      resources:
        limits:
          nvidia.com/gpu: 1
```

假定 CPU 等条件都满足。健康主例的数量候选是 `{gpu-h, gpu-a}`，型号候选是 `{gpu-h}`，两者的交集只有 `gpu-h`。这个交集表示必须同时满足的条件，实际框架可按配置执行各项过滤。

| 场景 | 型号候选 | 由余量得出的数量候选 | 放置结果 |
| --- | --- | --- | --- |
| A 已占用，B 健康，C、D 空闲 | `gpu-h` | `gpu-h` 有 1，`gpu-a` 有 2 | 可绑定 `gpu-h`。 |
| A、B 已占用，C、D 空闲 | `gpu-h` | 只有 `gpu-a` 有 2 | 无交集，继续等待可行节点。 |
| A 已占用，B 健康，C、D 空闲，新 Pod 改为请求两张 | `gpu-h` | 只有 `gpu-a` 有 2 | 无交集，`gpu-h` 的请求余量差一个单位。 |

## 混装节点的型号保证

另取一台 `gpu-mix`，装有 H100 X 和 A100 Y。两张卡都健康、未分配，并由插件上报为同一个 `nvidia.com/gpu` 资源。管理员给节点写上 `infra.example.com/has-h100: "true"`，表示节点里有 H100。若 Pod 改用这个标签并请求一个单位，调度器可以把它放到 `gpu-mix`。节点内实际可分配的清单仍包含 X 和 Y，因为分配时按容器请求的资源名找候选设备，两张卡都属于这个资源名。 [Kubernetes v1.36 节点选择](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) · [kubelet v1.36.0 按资源名分配设备](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/cm/devicemanager/manager.go)

主例的 H100 保证来自节点筛选与同型资源池的组合。回到 `gpu-h`，已经知道 Pod 会落在哪台节点，接下来可以从该节点的健康设备清单中追踪最终交付哪张卡。混装时若要在申请中直接表达逐卡条件，需要让调度器能读到每个设备自己的属性。
