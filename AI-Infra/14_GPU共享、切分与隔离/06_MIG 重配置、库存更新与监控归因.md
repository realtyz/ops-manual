# MIG 重配置、库存更新与监控归因

白昼线上流量高峰到来，业务急需把原本切给小模型测试的 `1g.5gb` 碎片回收，重新切成面向 14B 在线推理的 `3g.20gb` 硬件大实例。但 MIG 重配置绝不是改一个节点标签就万事大吉：一旦漏掉排空、跳过插件上报核对，或者拿重配置前的监控指标直接画连续曲线，轻则新 Pod 长期 `Pending`，重则监控面板把两套毫不相干的业务显存曲线拼成一场假告警。

本篇以双卡 `node-1`（配置见下表，`ClusterPolicy` 使用 `mixed` 策略）为例，完整走通“物理重切 → 库存收敛 → 监控跨生命周期归因”三步闭环：

| 设备与目标 | 变更前状态 | 变更后目标状态 | 说明 |
| --- | --- | --- | --- |
| `node-1` / `GPU 0` | `7 × 1g.5gb`（`seven-small`） | `2 × 3g.20gb`（`two-large`） | 已开启 MIG 模式，本次仅变更实例布局 |
| `node-1` / `GPU 1` | 整卡（关闭 MIG） | 整卡（关闭 MIG） | 全程保持 `nvidia.com/gpu: 1` 不变 |

## MIG 模式与实例布局的双层生命周期

管理 MIG 时必须先区分两层独立状态：**MIG 模式（Mode）** 决定物理卡是否允许切分，**实例布局（Layout）** 决定当前卡上实际存在哪些 GI 与 CI。

创建链路严格按顺序推进：`开启 MIG 模式 → 创建 GI → 在 GI 内创建 CI → CUDA 可枚举使用`。拆除链路则完全逆序：`摘流排空容器与持有设备的监控/管理进程 → 删除 CI → 删除 GI`。只要宿主机或容器里还有进程占用目标实例，布局变更就会被驱动拒绝。

> [!note] 硬件代际差异（Ampere vs Hopper）
> 在 A100（Ampere）上切换 MIG 模式时，驱动会尝试复位物理 GPU，且模式开关可以跨系统重启持久化；从 H100（Hopper）开始，切换 MIG 模式无需复位 GPU，但模式状态仅随驱动内核模块驻留，卸载重载驱动或重启后会恢复默认。**无论哪一代架构，已经创建的 GI/CI 实例布局在 GPU 复位或节点重启后都需要重新构建**（集群里通常由 MIG Manager 自动完成）。

## MIG Manager 驱动的布局重配置流程

在 `GPU Operator v26.7.2` 中，**MIG Manager（`v0.15.1`）** 持续监听节点上的 `nvidia.com/mig.config` 标签。管理员在集群中通过自定义 `ConfigMap`（由 `ClusterPolicy.spec.migManager.config.name` 引用）定义目标布局字典：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-mig-configs
  namespace: gpu-operator
data:
  config.yaml: |-
    version: v1
    mig-configs:
      seven-small:
        - devices: [0]          # 应用前须先用 UUID 与 PCI BDF 复核索引 0 确为目标卡
          mig-enabled: true
          mig-devices:
            "1g.5gb": 7
        - devices: [1]
          mig-enabled: false
      two-large:
        - devices: [0]
          mig-enabled: true
          mig-devices:
            "3g.20gb": 2        # 切换目标：两个 3g.20gb 硬件隔离推理实例
        - devices: [1]
          mig-enabled: false
```

正式执行变更前，必须先屏蔽调度并安全排空 `node-1` 上占用目标卡的在线推理副本，再切换节点标签触发重切：

```sh
# 1. 禁止新 Pod 调度到 node-1，并优雅排空运行中的工作负载（在线服务先完成摘流）
kubectl cordon node-1
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# 2. 将 MIG 配置标签从 seven-small 切换为 two-large
kubectl label node node-1 nvidia.com/mig.config=two-large --overwrite
```

标签切换后，节点侧依次走过如下状态收敛流转：

```mermaid
flowchart LR
    accDescr: 管理员排空节点并切换 nvidia.com/mig.config 标签后，MIG Manager 停止受管 GPU 客户端并重建 GI 与 CI 布局，随后恢复组件，Device Plugin 与 kubelet 刷新 Node 库存，最终由新 Pod 申请并完成 CUDA 探针验证。
    A["排空业务 Pod\n并切换 mig.config 标签"] --> B["MIG Manager 暂停受管客户端\n拆除旧实例并重建 GI/CI"]
    B --> C["恢复 Device Plugin/GFD/DCGM\n刷新 Node 扩展资源库存"]
    C --> D["取消封锁节点\n新推理 Pod 分配并过 CUDA 探针"]
```

*图 14-06-01：GPU Operator 托管节点执行 MIG 布局重配置与库存收敛的完整链路。*

实时跟踪 MIG Manager Pod 日志时，可以看到它依次打印出目标切换与恢复动作（涉及首次开关 MIG 模式且平台要求复位时，还可能触发节点重启）：

```text
# MIG Manager 典型收敛日志节选
Applying MIG device configuration...
Updating MIG config: map[3g.20gb:2]         # 正在按 two-large 模板应用新布局
Successfully applied MIG configuration
Restarting GPU clients (device-plugin, gfd, dcgm-exporter)...
Updating node label: nvidia.com/mig.config.state=success
```

## 四层收敛验收与只读核对命令

看到 MIG Manager 打印 `success`，只说明宿主机硬件切分动作已经完成，距离在线推理 Pod 真正可用还差两步：设备插件重新上报库存，以及容器端验证通过。上线解封（`kubectl uncordon node-1`）前，必须逐层通过四道验收关：

| 检查层 | 核对对象与通过标准 |
| --- | --- |
| **1. 管理器层** | `nvidia.com/mig.config` 为 `two-large`；状态标签 `nvidia.com/mig.config.state` 经由 `pending`（必要时经 `rebooting`）最终稳定在 `success`。 |
| **2. 宿主机硬件层** | `GPU 0` 上存在 2 个 `3g.20gb` GI 且各自挂有可用 CI；`GPU 1` 保持 MIG 关闭。 |
| **3. K8s 库存层** | Device Plugin、kubelet 与 GFD 重新上报完毕，Node `.status.allocatable` 精准反映新规格数量，旧规格清零。 |
| **4. 容器计算层** | 测试推理 Pod 申请 `nvidia.com/mig-3g.20gb: 1` 绑定成功，PodResources 返回新 MIG UUID，容器内成功枚举设备并跑通一次显存分配与小 Kernel 计算。 |

下面这组只读命令可一次性集齐宿主机硬件与 Kubernetes 控制面的全部证据：

```sh
# 在 node-1 宿主机侧：核对物理卡身份、MIG 模式、实例 UUID 与 GI/CI 占位
nvidia-smi --query-gpu=index,uuid,pci.bus_id,mig.mode.current --format=csv
nvidia-smi -L
nvidia-smi mig -lgi
nvidia-smi mig -lci

# 在集群控制面：核对节点状态标签与 Allocatable 库存是否收敛
kubectl get node node-1 -o yaml
```

对比变更前后 Node 对象上三种扩展资源的稳态库存：

| 稳定状态 | `nvidia.com/gpu` | `nvidia.com/mig-1g.5gb` | `nvidia.com/mig-3g.20gb` |
| --- | ---: | ---: | ---: |
| 重配置前（`seven-small`） | `1` | `7` | `0` |
| 重配置后（`two-large` 已收敛） | `1` | `0` | `2` |

表中的 `0` 在实际 Node YAML 中常表现为对应资源键直接消失。排查变更后的调度卡顿，抓住两条典型症状即可：

- **旧模板持续 `Pending`**：如果排空后某个旧 Deployment 依然申请 `nvidia.com/mig-1g.5gb: 1`，新布局下同名余量为 0，它自然会停在 `Pending`，必须同步更新业务清单。
- **宿主机已切好，Node 仍报旧键**：若 `nvidia-smi mig -lgi` 已显示两个 `3g.20gb` 且 `mig.config.state=success`，但 `kubectl get node` 依然停留旧库存，故障卡在**设备插件恢复、向 kubelet 重新注册或状态上报环节**，应立即排查 `nvidia-device-plugin` Pod 日志与重启状态。

## 跨生命周期监控归因：别被复用的 GI ID 骗了

实例切完并接入线上推理流量后，监控大盘里藏着一个极易踩中的归因陷阱。

DCGM Exporter 通过指标（例如 `DCGM_FI_DEV_FB_USED`，单位 `MiB`）持续采集显存用量，并在 MIG 实例级样本上附加 `GPU_I_ID`（GI 数字编号）与 `GPU_I_PROFILE` 等实体标签。要注意：**GI 数字编号在实例销毁重建后会被新实例复用，只有 MIG UUID 才在单次实例生命周期内保持唯一**。

看下面两条相隔 15 分钟、采自同一节点 `GPU 0` 的 GI 显存监控记录：

| 采集时刻 | Pod / 容器 | 分配的 MIG UUID | 宿主机映射 | GI 样本 `DCGM_FI_DEV_FB_USED` |
| --- | --- | --- | --- | --- |
| `10:00`（变更前） | `infer-prod/embed-svc` / `app` | `MIG-old` | `GPU 0` / `GI 2` / `CI 0` | `GPU_I_ID=2`, `GPU_I_PROFILE=1g.5gb`, `1,024 MiB` |
| `10:15`（变更后） | `infer-prod/llm-14b` / `app` | `MIG-new` | `GPU 0` / `GI 2` / `CI 0` | `GPU_I_ID=2`, `GPU_I_PROFILE=3g.20gb`, `4,096 MiB` |

如果你在 PromQL 里简单按 `(node, parent_gpu, GPU_I_ID="2")` 分组画图，图表会直接连出一条显存暴涨 $4,096 - 1,024 = 3,072\ \text{MiB}$ 的陡峭曲线，仿佛同一个服务发生了严重内存泄漏！

实际上，这两条样本分属**两个完全不同的实例生命周期**和**两套不同的业务 Pod**。正确的跨层归因顺序必须是：

$$\text{采样时刻} + \text{节点} + \text{父卡 UUID} + \text{GI 标签} \longrightarrow \text{当时有效的 MIG UUID} \longrightarrow \text{PodResources (Pod / Container / UID)}$$

> [!warning] 常见误区：同 GI 多共享与 Time-Slicing 下的监控盲区
> 1. 如果一个 GI 里建了多个 CI 或叠加了多个共享进程，GI 级 DCGM 样本给出的只是**父 GI 内的合计用量**，无法直接分摊给单个容器。
> 2. 在 `GPU Operator v26.7.2` 中，使用 NVIDIA Device Plugin 原生 **Time-Slicing** 时，DCGM Exporter **无法把 GPU 指标直接关联到单个容器标签**——此时只能借助 kubelet PodResources 确认哪几个容器同时持有该卡访问权，无法从单张物理卡指标反推某个容器独吞了多少显存或算力。

走完变更、库存收敛与跨周期监控归因，我们已经彻底摸清了各类切分池的脾气。下一篇把全章知识汇总成一张面向业务选型的决策树，并建立严谨的单跑与并跑验收方法。

> [!info]- 参考资料
> - **MIG 生命周期、代际差异与命令行参考**：[NVIDIA MIG 620 启用模式与实例管理](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/getting-started-with-mig.html)、[NVIDIA MIG 620 架构代际差异](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/getting-started-with-mig.html#enable-mig-mode)、[NVIDIA nvidia-smi 命令参考](https://docs.nvidia.com/deploy/nvidia-smi/index.html)
> - **GPU Operator v26.7.2 MIG 重配置与日志样例**：[GPU Operator v26.7.2 MIG 配置与管理流程](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-mig.html)、[GPU Operator v26.7.2 自定义 MIG ConfigMap](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-mig.html#example-custom-mig-configuration-during-installation)、[GPU Operator v26.7.2 重配置日志与验证](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-mig.html#example-reconfiguring-mig-profiles)
> - **PodResources 与 DCGM 指标归因边界**：[Kubernetes v1.36 PodResources API](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#monitoring-device-plugin-resources)、[DCGM Exporter 指标选择](https://docs.nvidia.com/datacenter/dcgm/latest/installation/install-dcgm-exporter.html#select-metrics)、[DCGM Exporter 实体标签规范](https://docs.nvidia.com/datacenter/dcgm/latest/reference/dcgm-exporter-metrics.html#exporter-labels)、[NVML 620 UUID 生命周期语义](https://docs.nvidia.com/deploy/nvml-api/620/api/group__nvmlDeviceQueries.html)、[GPU Operator v26.7.2 Time-Slicing 监控限制](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html#limitations)
