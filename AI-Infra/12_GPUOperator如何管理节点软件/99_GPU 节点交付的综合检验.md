# GPU 节点交付的综合检验

学完控制环路、init 启动门禁、Validator 边界、预装与离线制品链以及托管 DRA 后，我们用三组综合现场材料来检验整套排障与设计直觉。所有题目均可仅凭已有条件独立推导，建议先写下判断，再展开参考推理逐项核对。

## 基础集群环境与角色映射

主环境运行 **Kubernetes `v1.36`**、**GPU Operator `v26.7.2`**、**x86_64 Ubuntu 24.04 LTS** 与 **containerd `2.3.0`**，启用原生 CDI 与 Device Plugin，关闭 NRI 插件。节点构成如下：

| 语义角色 | 节点名 | 硬件规格 | 说明 |
| --- | --- | --- | --- |
| 普通 CPU 节点 | `cpu-0` | 无 GPU | 不匹配任何 GPU 组件模板 |
| 已有就绪节点 | `gpu-h0` | 2 × H100 PCIe（整卡独占） | 默认始终有 1 个就绪插件 Pod |
| 新入池目标节点 | `gpu-h1` | 2 × H100 PCIe（整卡独占） | 本次交付与排查的核心对象 |

---

## 现场材料一：新节点入池的三个观测时点

安装时的 Helm 值与线上 `ClusterPolicy.spec` 中，`clusterPolicy.deployCR`、`driver.enabled` 与 `toolkit.enabled` 均为 `true`。插件 DaemonSet 模板要求节点带 `nvidia.com/gpu.deploy.device-plugin=true`，且其余放置约束均已满足。

| 观测时点 | `gpu-h1` 节点与组件现场状态 | 插件 DaemonSet 期望 / 当前 / 就绪（节点） | `gpu-h1` GPU Capacity / Allocatable（整卡） |
| --- | --- | ---: | ---: |
| **时点 A** | Node 已 `Ready`，已有 NFD 的 NVIDIA PCI 标签，但缺少插件部署标签与插件 Pod；此刻 `ClusterPolicy` 汇总显示 `ready`。 | `1 / 1 / 1` | 两键缺失 |
| **时点 B** | 插件部署标签已变为 `true`；插件 Pod 已绑定 `gpu-h1`，但沙箱报错缺 `nvidia` handler；Toolkit 正等驱动，driver 主容器曾重启。 | `2 / 2 / 1` | 两键缺失 |
| **时点 C** | 两台 GPU 节点的插件 Pod 均已就绪；`gpu-h1` 已上报两张卡，但两张随后均被标记为不健康；集群使用默认 `plugin-validation`（检查 Node 容量）。 | `2 / 2 / 2` | `2 / 0` |

## 现场材料二：混合预装与隔离网络驱动切换（变体一）

以健康的两节点主环境为起点，运维策略发生以下变更：

| 变更维度 | 现场条件与目标计划 |
| --- | --- |
| **维护分工调整** | `gpu-h0` 改为宿主机预装驱动，`gpu-h1` 继续由 GPU Operator 托管驱动；两个节点池的 Toolkit 与 Device Plugin 仍由 Operator 统一部署。 |
| **离线当前故障** | `gpu-h1` 移入无公网的隔离网络，运行 Ubuntu 24.04 LTS，内核为 `6.8.0-52-generic`；传统 driver 镜像已从内网拉取成功，但主容器报错包仓库找不到匹配头文件。 |
| **后续切换计划** | 团队计划将 `gpu-h1` 切到 R595 预编译驱动模式，并同步将内核升级至 `6.8.0-53-generic`；目前内网镜像仓库仅同步了 `595-6.8.0-52-generic-ubuntu24.04`。 |

## 现场材料三：独立新集群的 `GPUCluster` 托管 DRA（变体二）

另有一套独立新建的集群（节点 OS、K8s 与 containerd 版本与主环境一致），通过 Helm 启用了 `GPUCluster/gpu-cluster`（关闭 ComputeDomain），并由 `NVIDIADriver` 交付驱动：
- `GPUCluster` 报告 `ready`，`gpu-h1` 上运行着 DRA 节点插件，并已发布包含两张整卡的 `ResourceSlice`。
- 普通业务 Pod 引用整卡类 `gpu.nvidia.com` 的 `ResourceClaim`，其 `status.allocation` 显示已分配 `gpu-h1` 上的一张卡。
- Pod 已绑定到 `gpu-h1`，DRA 节点插件的设备准备调用也返回成功，但 **containerd 创建容器时报错无法解析返回的 CDI 全名**。

---

## 检验任务

1. **跟踪覆盖与阻塞（材料一 A → B）**：说明从 A 到 B 哪一层输入变了，为什么期望数与当前数升至 `2` 而就绪数仍为 `1`？分别指出 A、B 最先该查哪项证据。
2. **拆解 Validator 盲区与余量复算（材料一 C）**：判断时点 C 默认 `plugin-validation` 能否通过，并复算新 Pod 调度余量。若稍后仅恢复一张卡且无已计入请求，推算新余量并说明下一步验收。
3. **划分维护责任与离线制品依赖（材料二）**：给出两池驱动配置方案；推导传统驱动当前缺少的包名与切换预编译后的目标镜像标签，并说明各自查证入口。
4. **定位托管 DRA 注入故障与迁移约束（材料三）**：指出哪些层级已有完成证据、最该查的下一环是什么；若把该 Helm 配置直接套用到已有 `ClusterPolicy` 的集群，首先违反哪项前提？

---

## 分步参考推理

> [!example]- 题 1 参考推理：用节点部署标签与 init 依赖解释 DaemonSet 计数变化
> - **时点 A 的判断与首查**：NFD 的 PCI 标签已到位，三个 `1` 全部来自旧节点 `gpu-h0`，`ClusterPolicy` 汇总 `ready` 此时仅反映旧节点状态，尚未覆盖 `gpu-h1`。最先应核对 **Operator 部署标签、节点排除设置与插件 DaemonSet 模板选择器**，确认新节点为何缺 `nvidia.com/gpu.deploy.device-plugin=true`。
> - **A → B 的输入变化**：集群策略开关没变，变的是**节点部署标签**（`gpu-h1` 变为 `true`）。新节点因此命中模板并建出插件 Pod，期望数与当前数同步升至 `2`。
> - **就绪数停在 1 的原因与首查**：插件 Pod 缺 `nvidia` handler 无法建沙箱，根因是 Toolkit 仍在等驱动就绪，而 driver 主容器已重启过。最先应读 `gpu-h1` 上 `nvidia-driver-ctr` 的 `--previous` 日志与内核日志，区分包安装还是模块加载失败。

> [!example]- 题 2 参考推理：破除默认容量探针盲区并复算调度余量
> - **默认插件校验结果**：时点 C 中 `Capacity = 2`，满足默认 `plugin-validation`（`WITH_WORKLOAD="false"`）的“容量至少为 1”，因此**默认检查仍可能通过**。
> - **时点 C 的调度余量复算**：两卡均不健康使 `Allocatable = 0`。无其他占卡请求时，调度余量为：
>
> $$\text{剩余可调度整卡数（时点 C）} = \text{Allocatable} - \text{已计入请求} = 0 - 0 = 0\text{ 张}$$
>
> - **单卡恢复后的状态**：仅恢复 1 张健康卡且 kubelet 写回后，`Capacity / Allocatable = 2 / 1`，调度余量为：
>
> $$\text{剩余可调度整卡数（单卡恢复后）} = 1 - 0 = 1\text{ 张}$$
>
> - **下一步验收**：提交带 `nodeSelector: kubernetes.io/hostname=gpu-h1`、申请 1 卡的普通 VectorAdd Pod。确认 `spec.nodeName=gpu-h1`，运行窗口内用 PodResources 核对设备 UUID，并以日志 `Test PASSED` 和退出码 `0` 确认计算通过。

> [!example]- 题 3 参考推理：混合池维护分工与隔离网络双制品链核查
> - **两池驱动方案**：保持全局 `driver.enabled: true`，仅在预装节点 `gpu-h0` 打单节点排除标签 `nvidia.com/gpu.deploy.driver=false`（禁用 `operands=false` 以免连 Toolkit 与插件一并排除）。这样 `gpu-h0` 驱动归节点团队，`gpu-h1` 归 Operator，两池共用托管 Toolkit 与插件。
> - **传统驱动当前缺包**：镜像已从内网拉通，问题在第二段包源。按运行内核 `6.8.0-52-generic`，缺少精确头文件包 **`linux-headers-6.8.0-52-generic`**。应查 `repoConfig` 包源配置、内网仓库索引与容器日志。
> - **预编译切换与升级**：内核升至 `6.8.0-53-generic` 后，R595 查询目标变为 **`595-6.8.0-53-generic-ubuntu24.04`**。现有 `-52-generic` 镜像对应旧内核，无法满足升级节点。切换前须核对 `v26.7` 预编译支持矩阵，并确认内网仓库已同步新标签与架构摘要。

> [!example]- 题 4 参考推理：锁定 DRA 最后一公里 CDI 解析与互斥前提
> - **已有完成证据的层级**：`GPUCluster` 就绪与 DRA 插件运行证明协调与覆盖；`ResourceSlice` 证明两卡库存发布；`ResourceClaim` 的 `status.allocation` 与 Pod 绑定证明选卡与落点；准备调用成功证明设备准备完成。
> - **最该查的下一环**：聚焦本机运行时 CDI 解析。把 DRA 插件返回的 CDI 全名，与 `gpu-h1` 上 containerd 生效的 `enable_cdi`、`cdi_spec_dirs` 及目录规格逐字比对（重发切片无效）。修通后用同形态 Claim 测试 Pod 验注入与 CUDA。
> - **套用到老集群的前提冲突**：首先违反**集群策略互斥与新装前提**。`v26.7` 中 `GPUCluster` 与 `ClusterPolicy` 不能并存，且托管 DRA 仅支持独立新装，不支持从已有 `ClusterPolicy` 原地迁移。

---

## 全章知识串联：节点软件如何托举集群算力

把第 12 章的七篇内容收束到同一条因果链上，无论是训练集群扩容还是推理池升级，所有运维与排障动作都落在这个骨架里：

1. **声明与纳管**：运维方借 Helm 与 `ClusterPolicy`（或新装 DRA 集群的 `GPUCluster`）声明期望软件栈。NFD 发现硬件，Operator 控制器打标维护模板，DaemonSet 把组件落到目标节点。
2. **门禁与解耦**：节点上靠 init 容器建立“驱动 → Toolkit → Validator → Device Plugin/GFD/Exporter”的启动秩序。混合池的维护责任则收口到全局开关、单节点排除、`/run/nvidia/driver` 契约与 `NVIDIADriver` CR。
3. **制品与验收**：离线与异构环境须分清镜像通道与内核包源通道（或预编译内核锁标签）。交付放行的唯一闭环标准，永远是业务形态 Pod 跑通调度、分配、CDI 注入与 CUDA 计算。

到这里，我们已经把 GPU 算力软件栈自动化交付给了 Kubernetes 节点。不过，多机训练的梯度同步与 PD 分离推理的跨机 KV Cache 迁移，光有 GPU 还不够，还需要 Pod 直接握住高速 RDMA 网卡。

[第 13 章 Multus 与 RDMA：把高速网络交给 Pod](../13_Multus与RDMA_把高速网络交给Pod/00_内容大纲.md)将把同一套分层交付与排障思路延伸到高速网络设备栈。

> [!info]- 参考资料
> - **GPU Operator 故障排查与安装场景**：[NVIDIA GPU Operator 故障排查](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/troubleshooting.html)、[NVIDIA 常用部署方式](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/getting-started.html#common-deployment-scenarios)
> - **离线、预编译与托管 DRA 参考**：[NVIDIA 离线镜像与包仓库配置](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/install-gpu-operator-air-gapped.html)、[NVIDIA 预编译驱动指南](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/precompiled-drivers.html)、[NVIDIA 托管 DRA 安装与验证](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html)
