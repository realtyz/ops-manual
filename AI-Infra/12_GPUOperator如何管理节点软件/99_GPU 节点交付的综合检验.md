
集群已有 `gpu-h0` 和 `cpu-0`，你负责加入 `gpu-h1`。两台 GPU 节点各有两张独占的 H100 PCIe。主例使用 GPU Operator `v26.7.2`、Kubernetes v1.36、x86_64 Ubuntu 24.04 LTS、containerd 2.3、CDI 和 Device Plugin，NRI 关闭。以下均为教学记录。用给出的材料先作判断，再与参考推理对照，无需运行命令或使用 GPU。

## 同一台节点的三个观测时点

安装时的 Helm 值中，`clusterPolicy.deployCR`、`driver.enabled`、`toolkit.enabled` 均为 `true`。线上 `ClusterPolicy.spec.driver.enabled` 和 `spec.toolkit.enabled` 同样为 `true`。插件 DaemonSet 模板要求 `nvidia.com/gpu.deploy.device-plugin=true`。假定 `gpu-h0` 始终有一个就绪插件，`cpu-0` 不匹配模板，其他 Pod 放置约束满足。

| 时点 | `gpu-h1` 的 Node 与组件 | 插件 DaemonSet 期望 / 当前 / 就绪节点数 | `gpu-h1` 的 GPU Capacity / Allocatable |
| --- | --- | ---: | ---: |
| A | Node 已 `Ready`，已有 NFD 的 NVIDIA PCI 标签。插件部署标签与插件 Pod 都缺失。此刻 `ClusterPolicy` 显示 `ready`。 | 1 / 1 / 1 | 两键缺失 |
| B | 插件部署标签已为 `true`。插件 Pod 已绑定 `gpu-h1`，其沙箱报告缺 `nvidia` handler。Toolkit 正等驱动，driver 主容器重启过。 | 2 / 2 / 1 | 两键缺失 |
| C | 两台插件 Pod 已就绪。`gpu-h1` 的 Device Plugin 已上报两张卡，两张随后都被标为不健康。默认的 `plugin-validation` 查看 Node 容量。 | 2 / 2 / 2 | 2 / 0 |

## 两种改变交付条件的材料

**变体一。** 以健康的两节点主例为起点。`gpu-h0` 预装驱动，`gpu-h1` 继续由 Operator 安装驱动，Toolkit 和 Device Plugin 仍由 Operator 为两个池提供。将 `gpu-h1` 移到隔离网络，其 Ubuntu 24.04 节点的运行内核为 `6.8.0-52-generic`。传统 driver 镜像已从内网拉取，主容器报告包仓库不存在匹配的头文件。计划切换到 R595 预编译模式时，内核也计划升到 `6.8.0-53-generic`。内网镜像仓库清单只有 `595-6.8.0-52-generic-ubuntu24.04` 对应的预编译镜像。

**变体二。** 在独立的新建集群采用 `GPUCluster/gpu-cluster`，节点版本与主例相同，ComputeDomain 关闭，已有 `NVIDIADriver` 交付驱动。`GPUCluster` 报告 `ready`，`gpu-h1` 有 DRA 节点插件与两张整卡的 `ResourceSlice`。普通 Pod 引用整卡类 `gpu.nvidia.com` 的 Claim，`status.allocation` 显示已分配该节点上的一张卡。Pod 已绑定，节点插件的准备调用成功。containerd 在创建容器时报告无法解析返回的 CDI 全名。

## 检验任务

1. 沿 A 到 B 说明哪一层的输入变了，为什么期望数和当前数增加，就绪数仍为 1。分别指出 A 和 B 最先查哪项证据。
2. 在 C 判断默认插件容量检查能否通过，并复算新 Pod 的 GPU 调度余量。若稍后仅恢复一张卡且没有已计入的 GPU 请求，推算新的容量与可分配量，说明如何继续验收。
3. 在变体一给出两池的驱动维护安排。根据运行内核推导传统驱动缺少的包和切换后的预编译标签，说明各自到哪里查证。
4. 在变体二指出哪几层已有完成证据，以及最该查的下一环。如果把这份 Helm 配置用到已有 `ClusterPolicy` 的集群，说明首先遇到哪项部署条件。

## 参考推理

### 1. 用节点覆盖解释 DaemonSet 计数

时点 A 的 PCI 标签已经有了，下一步对照 Operator 的部署标签、节点排除设置和实际 DaemonSet 模板。三个 1 都来自 `gpu-h0`，`ClusterPolicy` 的汇总无法证明 `gpu-h1` 已覆盖。到 B，策略保持为启用，改变的是节点的部署标签。`gpu-h1` 因此满足模板条件并有了插件 Pod，期望和当前从 1 变为 2。新插件缺少沙箱所需的 handler，使就绪数留在 1。Toolkit 还在等待驱动，尚不能配置运行时。先查 `gpu-h1` 上 `nvidia-driver-ctr` 的上一实例及内核日志，确定驱动卡在安装还是加载。

### 2. 用健康上报计算调度余量

时点 C，`Capacity=2` 已满足默认 `plugin-validation` 的至少一张卡的容量检查。`Allocatable=0`，所以在没有其他请求的条件下，调度余量是 `0 − 0 = 0`。若一张卡恢复健康且 kubelet 已写回状态，则 `Capacity/Allocatable=2/1`，剩余一张。此时提交一个带目标节点选择器、申请一张 GPU 的普通 VectorAdd Pod。确认它被调度到 `gpu-h1`，用 PodResources 核对设备分配（需要在运行窗口取证），用 `Test PASSED` 和退出码 0 确认 CUDA 计算。这一次通过的范围是所测节点、设备与镜像。

### 3. 从驱动维护方追到制品仓库

应保留全局 `driver.enabled=true`，只在 `gpu-h0` 使用 `nvidia.com/gpu.deploy.driver=false`，由节点团队维护其驱动。`gpu-h1` 仍由 Operator 管理，两个池继续部署 Toolkit 与设备插件。传统 driver 镜像既已拉到，先查其包源配置和日志，确认能否取到 `linux-headers-6.8.0-52-generic` 及其他所需依赖。换内核后，预编译模式应查 `595-6.8.0-53-generic-ubuntu24.04`。用支持矩阵核对 OS、内核变体和分支，再查内网镜像仓库是否有这个完整标签和目标架构的摘要。已备的 `-52-generic` 镜像对应旧内核。

### 4. 用 Claim 和容器事件定位 DRA 注入

`GPUCluster` 状态与目标节点插件表明协调与覆盖，ResourceSlice 表明已发布库存，Claim 的分配和 Pod 绑定表明所用设备与节点已确定。准备调用返回后，containerd 还要从本机 CDI 规格解析名称。比较插件返回的全名与运行时扫描目录里的规格，确认 `enable_cdi`、`cdi_spec_dirs` 和实际文件，再用相同形态的 Claim 消费者验证注入与 CUDA。若集群已有 `ClusterPolicy`，则不满足 `GPUCluster` 托管路径的集群前置条件。v26.7 的托管流程面向新装，两个策略不能并存。

## 节点软件如何支撑资源交付

把安装和使用放回同一张关系图，故障就可以落到一项具体的输入、执行者或输出。安装方交付策略和制品，控制器据此部署节点组件。组件准备驱动、资源库存和运行时，Kubernetes 再用库存放置工作负载，最终让进程使用设备。更换节点镜像、驱动制品或分配后端时，要从改变的输入起顺着这条关系验收。第 13 章将把同一种思路应用于高速网络交付，增加 NIC 驱动、附加网络和 Pod 内的数据路径。
