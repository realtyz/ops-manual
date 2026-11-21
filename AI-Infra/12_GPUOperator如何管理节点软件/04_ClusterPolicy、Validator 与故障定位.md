# ClusterPolicy、Validator 与故障定位

排查新节点拿不到 GPU 时，最容易让人误判的现象莫过于：`ClusterPolicy` 显示 `ready`，Device Plugin DaemonSet 也是 `1/1 Running`。可新上架的节点上偏偏没有 `nvidia.com/gpu` 资源。

出现这种反差时，切忌直接扎进某个容器日志。GPU 节点排障必须遵循**由外向内**的顺序：**先看新节点进没进部署范围，再看组件 Pod 卡在哪道 init 门禁，最后才深挖主容器日志与运行时注入**。

## 别被集群级 `ready` 迷惑：先锁定节点覆盖范围

`ClusterPolicy.status.state` 只是控制器最近一次集群级协调的汇总。在 `v26.7.2` 中，哪怕集群还没采集到 NFD 标签，或者有标签但未匹配到任何 GPU 节点，该字段依然可能报 `ready`。

要看清控制器的真实判断，必须读 `status.conditions[]` 里的 `type`、`status`、`reason` 和 `message`。借此可以明确区分 `NFDLabelsMissing`、`NoGPUNodes` 与 `Reconciled` 等状态原因。

想进一步追到具体组件，可直接检索控制器日志里的 `ClusterPolicy step completed` 及其 `state`、`status` 字段。

确认策略生效后，接着核对目标组件 DaemonSet 的三个核心计数：
- **`desiredNumberScheduled`**：满足模板选择器的目标节点总数。
- **`currentNumberScheduled`**：其中已建出该 DaemonSet Pod 的节点数。
- **`numberReady`**：其中已有就绪 Pod 的节点数。

若这三项都是 `1` 且唯一 Pod 跑在已有节点上，说明新节点还没进入模板范围。此时应直接比对 DaemonSet 的选择器、亲和性、容忍与新节点的标签和污点。只有当 `desired` 升至 `2` 而 `current` 仍为 `1` 时，才需要追查 Pod 创建与调度事件。

![Device Plugin DaemonSet 的部署范围仅覆盖已有节点 gpu-h0，新节点 gpu-h1 已发现 NVIDIA PCI 但缺少部署标签与插件 Pod](images/fig-12-04-01-daemonset-scope.png)

*图 12-04-01：集群级 ready 与新节点未覆盖的典型反差。图中 ClusterPolicy 显示 ready、插件 DaemonSet 期望与就绪均为 1，实际仅覆盖已有节点 gpu-h0；新节点 gpu-h1 虽已发现 NVIDIA PCI，但缺少 nvidia.com/gpu.deploy.device-plugin=true 标签，尚未进入部署范围。*

确认目标节点上已有组件 Pod 后，按 `spec.nodeName` 与 `metadata.uid` 把 Pod、事件与日志对齐。查看 `status.initContainerStatuses` 可直接锁定卡住的 init 阶段。遇到反复重启的容器，务必带 `--previous` 读取上一实例日志：

```bash
# 1) 查看 ClusterPolicy 详细状态条件与控制器最新协调原因
kubectl get clusterpolicy cluster-policy -o jsonpath='{.status.conditions}'

# 2) 查看目标节点组件 Pod 的 init 状态与事件（<pod> 替换为实际 Pod 名）
kubectl describe pod -n gpu-operator <pod>

# 3) 读取已重启容器的上一实例带时间戳日志（如驱动主容器 nvidia-driver-ctr）
kubectl logs -n gpu-operator <pod> -c <container> --previous --timestamps
```

> [!tip] 工程直觉
> 排障三步铁律：**先查“进没进范围”（DaemonSet 期望数与节点标签），再查“卡在哪道门禁”（init 状态与事件），最后查“容器里报什么错”（`--previous` 日志与内核日志）**。跳过第一步直接看日志，极易看错节点。

## Validator 四道关卡与 `Capacity / Allocatable` 盲区

在默认配置下，每个纳入范围的 GPU 节点都会运行 `nvidia-operator-validator` Pod，按序执行四项 init 校验。把每一关的输入、产物与排障抓手收进一张状态表，就能一键定位阻塞环节：

| init 校验阶段 | 实际检查动作与产出 | 卡住/失败时的典型根因 | 首选排查证据 |
| --- | --- | --- | --- |
| `driver-validation` | 检查宿主机驱动，必要时校验容器化驱动；通过后写入 `driver-ready` 标记 | 驱动未加载、模块编译失败或驱动根目录不匹配 | 读 `driver-ready` 中的 `IS_HOST_DRIVER` 与 `NVIDIA_DRIVER_ROOT`，查 driver Pod 与内核日志 |
| `toolkit-validation` | 在 GPU 管理容器中执行 `nvidia-smi`；通过后写入 `/run/nvidia/validations/toolkit-ready` | `nvidia` handler 未生效、Toolkit 配置未指向目标 containerd | 查管理容器沙箱事件与 Toolkit Pod 日志 |
| `cuda-validation` | 默认开启工作负载检查，创建显式指定 `spec.nodeName` 的 CUDA 测试 Pod | 容器化 CUDA 初始化失败或驱动与计算库不兼容 | 查验证 Pod 的容器日志与终止退出码 |
| `plugin-validation` | 默认 `WITH_WORKLOAD="false"`，仅检查 Node 的 `status.capacity["nvidia.com/gpu"] >= 1` | Device Plugin 未注册成功或设备枚举为空 | 查插件注册日志及 Node `Capacity` / `Allocatable` 字段 |

这里藏着一个经典的**假绿盲区**。假设新节点上的两张卡都在设备列表中，却全部被标记为不健康。按 Kubernetes 的计数规则，节点会呈现 **`Capacity = 2`、`Allocatable = 0`**。哪怕节点上没有其他业务占卡，调度余量也只有：

$$\text{剩余可调度整卡数} = \text{Allocatable} - \text{已计入 GPU 请求} = 0 - 0 = 0\text{ 张}$$

然而，默认的 `plugin-validation` 只读 `Capacity`（此时仍为 `2 >= 1`），因此**插件校验照样绿灯通过**。此外，`cuda-validation` 创建验证 Pod 时直接写死了 `spec.nodeName`，绕开了 kube-scheduler 的常规调度决策。

> [!warning] 常见误区：Validator 全绿 ≠ 业务 Pod 能调度运行
> 默认 Validator 通过只说明节点已有驱动、管理容器可跑、API 上有容量数字。要确认节点能接住真实调度请求，必须核对 `Allocatable > 0` 与设备健康状态，并用普通 `nodeSelector` 业务 Pod 走完闭环。

如果想让 Validator 在插件阶段额外跑一个真正申请 GPU 的测试负载，可在 Helm 值里把 `WITH_WORKLOAD` 显式设为字符串 `"true"`。该测试 Pod 依然指向本节点，但其 init 容器会真实申请 1 张可用卡：

```yaml
# Helm values.yaml：开启 plugin-validation 的真实 GPU 申请测试
validator:
  plugin:
    env:
      - name: WITH_WORKLOAD
        value: "true"        # 必须为字符串 "true"；开启后要求节点至少有 1 张 Allocatable 整卡
```

## 分层故障现象与首选取证清单

把整条交付链的典型故障按“从底向上”的层次归类。现场遇到异常时，按已见证据直接对号入座，就能精准锁定下一项取证：

| 现场已见证据 | 故障收口层次 | 第一项精准取证动作 |
| --- | --- | --- |
| Toolkit 卡在 `driver-validation`，driver 主容器曾重启；插件 Pod 停在 `toolkit-validation` | 宿主机驱动层（上游根因，下游均为连锁等待） | 读 `nvidia-driver-ctr` 的 `--previous` 日志与宿主机内核日志，区分包安装失败还是模块加载失败；若 driver Pod 自身卡在 init，查 `k8s-driver-manager` |
| Device Plugin Pod 已 `Ready`，但 Node 的 `Capacity` 与 `Allocatable` 均缺少 GPU 键 | 设备枚举与 kubelet 注册层 | 查插件日志中的 NVML 初始化、设备枚举与注册结果，对照 kubelet 接收列表与 Node 状态写回时间 |
| 管理 Pod 报 `FailedCreatePodSandBox`，消息含 `no runtime for "nvidia" is configured` | 容器运行时 handler 接入层 | 核对 RuntimeClass 的 handler 与 kubelet 实际连接的 containerd；查 Toolkit 日志及 `CONTAINERD_CONFIG`、`CONTAINERD_SOCKET` 环境变量，确认目标实例已加载配置 |
| `Capacity / Allocatable` 为 `2 / 2`，普通 Pod 已绑定目标节点，但应用容器卡在创建阶段 | kubelet 分配或 CDI 注入层 | 查 kubelet `Allocate` 记录；确认已返回 CDI 全名后，检查 containerd 是否扫描到对应的 CDI 规格文件 |
| 业务测试容器已启动，随后非零退出 | 容器内 CUDA 初始化或计算层 | 对照分配的设备 UUID、容器内可见设备与 CUDA 报错日志，区分初始化失败还是 Kernel 执行失败 |

> [!summary] 核心结论
> 高效排障靠**分层证据替代猜测**。先用 DaemonSet 计数验覆盖，再用 init 门禁和验证标记锁驱动与运行时。接着用 `Allocatable` 破除 Validator 容量盲区，最后用普通 Pod 验注入与计算。

还有一种常见陷阱：明明按流程找 driver Pod，节点上却根本没有它，或者 Toolkit 始终找不到驱动文件。这往往源于**宿主机预装组件与 Operator 托管组件的维护责任边界发生了变化**。下一篇我们专门讲透预装模式与多版本 `NVIDIADriver` 的治理规则。

> [!info]- 参考资料
> - **ClusterPolicy 协调与 DaemonSet 状态**：[v26.7.2 ClusterPolicy 收尾分支](https://github.com/NVIDIA/gpu-operator/blob/v26.7.2/controllers/clusterpolicy_controller.go#L195)、[Kubernetes DaemonSetStatus](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/apps/daemon-set-v1/#daemonsetstatus)、[插件 DaemonSet 的节点选择](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/assets/state-device-plugin/0500_daemonset.yaml)
> - **Validator 实现与容量检查源码**：[v26.7.2 Validator 模板](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/assets/state-operator-validation/0500_daemonset.yaml)、[v26.7.2 Validator 实现](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/cmd/nvidia-validator/main.go)、[Validator 容量检查](https://github.com/NVIDIA/gpu-operator/blob/v26.7.2/cmd/nvidia-validator/main.go#L1365)、[验证 Pod 的 GPU 请求](https://github.com/NVIDIA/gpu-operator/blob/v26.7.2/cmd/nvidia-validator/main.go#L1266)、[ClusterPolicy 模板](https://github.com/NVIDIA/gpu-operator/blob/v26.7.2/deployments/gpu-operator/templates/clusterpolicy.yaml#L82)、[验证环境变量转换](https://github.com/NVIDIA/gpu-operator/blob/v26.7.2/controllers/object_controls.go#L2299)
> - **故障排查与设备插件规范**：[NVIDIA GPU Operator 故障排查](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/troubleshooting.html)、[NVIDIA Operator 的 Init 排查](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/troubleshooting.html#gpu-operator-pods-are-stuck-in-init)、[kubectl logs](https://v1-36.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)、[Kubernetes Device Plugin 接口](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)、[Kubernetes 不健康设备的资源计数](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#device-plugin-and-unhealthy-devices)
