# 从 kubelet 分配走到容器交付

## scheduler 绑定节点，Device Manager 确定 ID

沿用场景 P 的全健康快照：gpu-a 的 U0/U1 已分配，U2/U3 空闲。现在 `probe-p/worker` 请求一张卡，并且调度器已选 gpu-a。**绑定**表示 Pod 的 `spec.nodeName` 被设为该节点；在传统扩展资源路径中，Pod spec 的 `nvidia.com/gpu: 1` 仍没有写入 U2 的 UUID。

kubelet 接手该 Pod 的节点内准入与启动。Device Manager 从该资源的健康设备中排除已分配 ID，得到本轮候选 `{U2,U3}`，再决定其中一个。插件若声明支持 `GetPreferredAllocation`，可以根据候选集合、所需数量等返回建议；最终选择由 Device Manager 负责，建议并非最终分配的保证。若配置了节点内资源对齐，设备拓扑还会参与相关决策。这里固定其他约束已满足，**另给定本轮确实选中 U2**，不从 ID 大小推断默认算法会选编号最小的卡。

随后 kubelet 用具体 ID 调用插件的 `Allocate`。接口名含“分配”，容易让人误以为调度器将“1 张”的请求直接交给插件随便挑；实际上 `ContainerAllocateRequest.devices_ids` 已列出 Device Manager 选定的设备。协议与 NVIDIA 处理可分别查 [1.34 api.proto 的 AllocateRequest](https://github.com/kubernetes/kubernetes/blob/v1.34.0/staging/src/k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1/api.proto)和 [0.17.1 server.go 的 Allocate](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.17.1/internal/plugin/server.go)。

## Allocate 返回创建所需信息

主线明确选择 NVIDIA 插件 `cdi-cri` 策略、UUID 设备 ID，无共享/MIG。下面是**插件配置片段**，不是 DaemonSet、Helm values 或可独立部署的清单：

```yaml
version: v1
flags:
  migStrategy: none
  plugin:
    deviceListStrategy: cdi-cri
    deviceIDStrategy: uuid
    passDeviceSpecs: false
```

下面的 **RPC 教学摘要**保留协议字段名，使用 JSON 形式便于阅读，不是网络抓包或完整 protobuf 转储：

```json
{
  "AllocateRequest": {
    "container_requests": [
      {"devices_ids": ["GPU-33333333-3333-4333-8333-333333333333"]}
    ]
  },
  "AllocateResponse": {
    "container_responses": [
      {"cdi_devices": [
        {"name": "nvidia.com/gpu=GPU-33333333-3333-4333-8333-333333333333"}
      ]}
    ]
  }
}
```

输入是 U2 的设备 ID；输出是运行时应消费的完整 CDI 名称。两者恰好都含 UUID，但属于不同命名空间。前者参与本节点资源分配，后者匹配 CDI 描述并展开创建修改；不能把字符串相似当成同一个协议字段。

通用 `AllocateResponse` 可以包含环境变量、挂载、设备节点、注解和 CDI 设备名；不同实现/策略只用其中相关部分。**响应列出了某项修改，仍要由后续创建流程执行。**本例 `passDeviceSpecs: false`，不会因此缺失 GPU：实际设备节点、驱动文件和访问规则由 CDI 描述及其消费者落实，不能要求所有方案同时填满每个响应字段。

`PreStartContainer` 是设备插件可选 RPC，供实现进行设备特定的启动前准备；它不是 OCI prestart Hook，也不能因名字相近而把第 10 章 NVIDIA legacy Hook 填进这条调用记录。本例不依赖这一可选 RPC。

## 把同一 U2 送到运行时

下面是**自拟成功时间线**。每行是责任阶段，不规定 kubelet 内部所有调用的精确先后：

| 阶段 | 本轮可观察事实 | 由此得到的结论 |
| --- | --- | --- |
| 节点选择 | `probe-p.spec.nodeName=gpu-a` | Pod 已绑定，不证明设备已交付 |
| 节点内分配 | `probe-p/worker -> U2`；Allocate 成功 | 本次容器关联到具体 ID，已得到接入信息 |
| CRI 创建请求 | `CDIDevices` 含上述 U2 完整 CDI 名称 | 选择已向支持 CDI 的 CRI 路径传递 |
| containerd 原生 CDI | 名称匹配有效描述，源文件和 Hook 可用 | 设备及驱动相关修改能合并进创建配置 |
| 创建完成 | U2 的 `/dev/nvidia2` 与公共接口可访问 | 进程具备本轮设备访问条件 |
| 应用探针 | `cuda_count=1; cuda:0_uuid=U2; sync=ok; verify=PASS` | 本轮计算确实在 U2 完成并通过结果检查 |

`CDIDevices` 是 CRI 请求层面的字段称呼，不能原样当作 Pod spec 字段。Kubernetes 1.34 已支持设备插件响应中的 CDI 名称传递；还需 runtime 真的支持并启用 CDI、搜索目录内有相符描述。containerd 1.7.27 的 `enable_cdi=true` 条件沿用[第 10 章 CDI 篇](../10_GPU如何进入容器/03_用CDI把设备名变成运行环境.md)。

本轮最关键的对应是：**资源请求 1 → 节点 gpu-a → 插件 ID U2 → CDI 名称 → 进程中的 cuda:0（UUID U2）**。

<!-- 配图 fig-11-03-01：置于本段后，突出节点选择、具体身份、运行环境三个责任边界。 -->

CUDA ordinal 0 是进程只看到一张卡后的本地枚举，不会把主机 `/dev/nvidia2` 重命名成 `/dev/nvidia0`。仍应靠 UUID 把进程视角对应回设备。应用探针的同步与数值验证比只看管理查询多覆盖了实际计算，但本轮 PASS 仍没有测试 NCCL 或 RDMA。

## envvar 是另一条交付路径

NVIDIA 0.17.1 还提供 `envvar`、`volume-mounts`、`cdi-annotations` 等策略。策略决定**如何传递设备选择**，不会把 scheduler 变成节点内 UUID 分配者。完整选项见 [v0.17.1 Configuration Option Details](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.17.1/README.md#configuration-option-details)。本章只对照以下两条：

| 独立方案 | Allocate 的关键结果 | 后续消费者与条件 |
| --- | --- | --- |
| 本章主线 `cdi-cri` | `cdi_devices` 中的完整名称 | kubelet/CRI 传递；containerd 已启用原生 CDI；有效设备描述 |
| 对照 `envvar` | `NVIDIA_VISIBLE_DEVICES=U2完整UUID` | 已选 NVIDIA handler；本例显式 legacy 模式解析变量并调用注入逻辑 |

envvar 对照沿用第 10 章固定 legacy 的条件，并要求镜像功能选择含 `compute,utility`。不能把 Toolkit 1.18.0 的自动模式一律说成 legacy。单纯改变插件策略而不改变对应消费者，会出现“资源分配成功、GPU 没进容器”。同样，原生 `cdi-cri` 路径使用普通低层运行时并不自动构成故障。

## 从哪里读取分配到了谁

Pod spec 的数字不足以对应 UUID。kubelet 提供节点本地 **PodResources API**，其中传统设备的 `resource_name` 与 `device_ids` 可对应 Pod、容器和设备。默认 socket 是 `/var/lib/kubelet/pod-resources/kubelet.sock`；这是本地 gRPC 服务，不是 `kubectl get` 的 REST 资源，也不要求无环境读者自行部署客户端。

下面是**假定由兼容客户端调用 List 后得到的自拟 JSON 摘要**，省略 CPU/内存项；字段依据 [v1.34.0 PodResources api.proto](https://github.com/kubernetes/kubernetes/blob/v1.34.0/staging/src/k8s.io/kubelet/pkg/apis/podresources/v1/api.proto)：

```json
{
  "pod_resources": [{
    "name": "probe-p", "namespace": "lesson-dp",
    "containers": [{
      "name": "worker",
      "devices": [{
        "resource_name": "nvidia.com/gpu",
        "device_ids": ["GPU-33333333-3333-4333-8333-333333333333"]
      }]
    }]
  }]
}
```

现在可把 `probe-p/worker`、U2 和容器探针中的 UUID 对齐。PodResources 证明的是 kubelet 的分配视图；它自身不执行容器计算测试。

设备分配需要随容器重启保持一致的节点内记录，kubelet 会维护相关检查点和已分配集合。重启同一容器不能简单视为申请另一份 GPU；Pod 被删除、节点完成相应清理后，设备才重新进入可分配集合。Device Plugin 协议没有与 Allocate 对称的通用 `Deallocate` RPC，不能在时序图里凭空添加一次调用；应用结束、Pod 生命周期清理与设备健康也需分别判断。

[上一篇：用数量请求判断 Pod 能放在哪里](02_用数量请求判断Pod能放在哪里.md) · [下一篇：用 NFD 与 GFD 补充节点属性](04_用NFD与GFD补充节点属性.md)
