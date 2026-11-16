# GPU 管理权限、设备访问与租户隔离

上一篇确定了由谁维护节点配置，接下来要确定谁有权使用和改变它。这里先在 B 域已验收的节点上准备两个测试租户，用小规模探针验收权限；它们不是两组同时占满三节点的生产训练。两个租户可使用池中的不同整卡，允许各自成员通信，要求默认网和 RoCE 附加网都阻断指定的跨租户通信。h8 交付时要继承同一套经过验证的规则。

**租户隔离**包含三个可观察的边界：身份能修改哪些平台对象，容器实际能访问哪些设备，报文能通过哪些网络执行点。命名空间用于组织对象；只有把对象权限、Pod 准入和真实访问路径连接起来，才能把“属于不同租户”变成可执行的限制。

## 一次租户请求经过哪些权限边界

先走通正常请求。租户甲提交一个请求一张 GPU 的 Pod，API 服务检查身份能否创建它，准入机制检查模板是否符合规则；调度和 kubelet 为其选择节点与具体设备，运行时注入这张设备，网络组件为其建立允许的接口。容器随后能计算，并能与同租户对端通信。每一步输入不同，因此需要不同证据。

| 检查位置 | 要回答的问题 | 本章采用的验证方式 |
| --- | --- | --- |
| API 授权 | 该身份能否修改 Operator、节点标签或管理工作负载 | 用实际租户身份检查所需操作允许、越权修改拒绝 |
| Pod 准入 | 允许创建的 Pod 能否请求宿主权限、管理运行时或验收容忍 | 提交正常与违规模板，核对拒绝原因及最终 Pod |
| 设备交付 | 容器拿到的设备是否与本次分配一致 | 关联 Pod UID、PodResources 设备 ID 和容器内 UUID |
| 网络执行 | 接口已存在之后，哪些报文能到达对端 | 对指定接口与协议做同租户成功、跨租户拒绝对照 |

API 拒绝发生在对象被接受之前，网络拒绝发生在传输路径上。两者都重要，但不能用一个来证明另一个。

## GPU 管理组件的主机权限与准入控制

读第 01 篇的配置归属时，沿 GPU Operator 生成的容器追踪职责。Toolkit 要配置节点容器运行时，特定管理组件要访问宿主的文件、进程或设备。以 v26.7.2 的 Toolkit DaemonSet 模板为例，检查 `hostPID`、容器 `securityContext`、主机挂载和 ServiceAccount，再对照实际生成的对象。Operator 的 API 权限可让它创建或更新组件，运行中容器对主机的权限由 Pod 配置、挂载和运行时共同决定。管理组件镜像与配置属于应受平台管理和审计的交付输入。 [NVIDIA · GPU Operator 安全说明](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/security.html) · [v26.7.2 Toolkit DaemonSet 模板](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/assets/state-container-toolkit/0500_daemonset.yaml)

Kubernetes 的 **RBAC**（基于角色的访问控制）决定某个身份可对哪些 API 对象执行什么操作。**Pod Security Admission**（PSA，Pod 安全准入）依命名空间设置检查 Pod 是否符合相应安全标准。本例平台在独立的管理命名空间部署所需的高权限组件，只让运维自动化和指定管理身份能在那里创建或修改工作负载。还要核对租户能否修改命名空间的安全标签、平台 ServiceAccount 上的权限绑定和节点池策略。对租户命名空间，采用满足作业要求的受限策略并检查模板最终生成的 Pod，例如要求拒绝 `privileged` 和宿主路径挂载。管理命名空间允许高权限模板通过准入，现实中能否创建这类 Pod 继续受 API 权限约束。 [Kubernetes v1.36 · RBAC 最佳实践](https://v1-36.docs.kubernetes.io/docs/concepts/security/rbac-good-practices/) · [Kubernetes v1.36 · Pod 安全标准](https://v1-36.docs.kubernetes.io/docs/concepts/security/pod-security-standards/) · [NVIDIA · Operator 的 PSA 部署前提](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/getting-started.html#prerequisites)

例如租户有创建 Pod 的权限，并不表示它应有挂载宿主根目录的权限。RBAC 在这里已经允许“创建 Pod”，后者必须由准入策略审查具体字段。同样，能修改负责写入这些字段的控制器模板，也可能间接获得相同能力，因此需同时保护管理工作负载、ServiceAccount（工作负载的 API 身份）和权限绑定。第 01 篇要求一个组件有清楚的配置归属，本篇进一步要求只有相应管理身份能改变该来源。

## 整卡分配、配额与运行时访问

普通训练 Pod 请求 `nvidia.com/gpu`，由 Device Plugin 与 kubelet 完成分配，再通过 CDI 把设备交给运行时。按 PodResources 和容器内 UUID 验证两租户实际使用不同物理卡。本例可用命名空间 ResourceQuota 中的 `requests.nvidia.com/gpu` 限制整卡总请求，并用队列策略控制组的准入。配额约束的是 Kubernetes 接受的请求，若设备交付存在另外的访问路径，仍须检查该路径的准入和运行时。 [Kubernetes v1.36 · 扩展资源的配额](https://v1-36.docs.kubernetes.io/docs/concepts/policy/resource-quotas/#quota-for-extended-resources) · [Kubernetes v1.36 · Device Plugin 的分配与监控](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

例如 GPU Operator v26.7 的 CDI 模式下，NRI 关闭时，以 `NVIDIA_VISIBLE_DEVICES` 请求直接访问设备的 GPU 管理容器使用 `runtimeClassName: nvidia`。该路径由平台管理。实施租户边界时，应在租户准入中限制可用的 RuntimeClass 及管理式设备访问配置，再用实际容器确认访问范围。PSA 的安全标准并未为这个 NVIDIA 路径提供专门的 RuntimeClass 策略。不同物理 GPU 仍可能共用节点、驱动、供电和网络。若需求是节点级故障隔离，应加放置约束。若改成第 14 章的时分共享，还需另行核对显存可用空间与故障传播范围。 [NVIDIA · CDI 下的 GPU 管理容器](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/cdi.html#cdi-and-gpu-management-containers) · [NVIDIA · GPU 时分共享的隔离边界](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html)

RuntimeClass 把 Pod 中的运行时名称对应到节点上的处理方式；它本身不是“这只 Pod 已获准使用几张 GPU”的凭据。正常整卡请求应沿设备分配返回的 CDI 引用完成交付。若另有管理访问路径可让租户指定更多设备，配额只统计到声明的请求，便无法代表真实访问范围。

可以用两个条件变体检查这一点：把租户配额由 4 卡降到 2 卡，改变的是新增请求是否被接受；把租户 Pod 的运行时策略改成允许管理式设备访问，改变的是运行时可能授予的能力。前者不能补救后者。正式验收既要看到合规的一卡请求只获得本次分配，也要看到越界模板被拒绝、未分配设备无法通过其他允许路径访问。若需求涉及跨租户故障隔离，还须另查共用驱动、主机与互联的影响域。

## 默认网络与 RoCE 附加网络的隔离验收

第 13 章中，默认接口服务于普通 Pod 通信，附加接口把 SR-IOV VF 等网络能力交给 Pod。Multus 负责委派网络配置，NAD（NetworkAttachmentDefinition，附加网络定义）描述要接入的网络。**允许引用某个 NAD**解决“能接哪张网”，**数据面隔离**解决“接入之后能和谁通信”。VF 是独立分配单位，但多个 VF 仍可能共享物理端口或同一转发域。

默认 Pod 网络由实现 Kubernetes NetworkPolicy 的网络插件执行策略。Multus v4.2.4 开启命名空间隔离后会检查 Pod 可引用的 NAD，还应核对允许跨命名空间引用的例外。RoCE 附加网络使用映射到 PF 的 VF 通信，执行边界由网络设计决定。例如可把两个租户放在经平台验证互不可达的网络段，并在实际转发位置实施隔离。对于在硬件内转发的报文，需要确认执行点覆盖该路径。策略对象和附加网接入记录只说明规则已声明、接口已交付，数据面的拒绝需要实测。 [Kubernetes v1.36 · NetworkPolicy 的前提和范围](https://v1-36.docs.kubernetes.io/docs/concepts/services-networking/network-policies/) · [Multus v4.2.4 · NAD 命名空间隔离](https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/v4.2.4/docs/configuration.md)

以下是开通第二租户前的隔离验收记录。测试在授权的测试租户和预先部署的对端进行，同租户允许与跨租户拒绝使用相同测试应用与受控条件。按网络固定源与目的接口、地址、传输协议、监听或 RDMA 接收条件，并用数据面执行点的日志或计数判读策略是否覆盖实际路径。

| 路径 | 期望允许的同租户试验 | 期望拒绝的跨租户试验 | 本轮结论 |
| --- | --- | --- | --- |
| 默认网 TCP | 连接成功，服务响应 | 连接被默认网络执行点拒绝，命中相应规则 | 本次 TCP 对照覆盖两个指定端点。 |
| 附加网 RoCE | 使用指定 VF 的双端 RDMA 测试成功 | 启用了仅作用于附加网 TCP 的规则。双端 RoCE 验证尚未执行 | 有附加网正常传输证据，跨租户的 RoCE 拒绝待验证。 |
| 引用他租户的 NAD | 本租户 NAD 能接入 | 他租户 NAD 被接入校验拒绝 | 附加网选择的边界已验证。 |

![甲乙两个测试 Pod 各有默认接口和附加接口。上方默认网 TCP 的测试箭头停在策略执行点，旁边记录跨租户 TCP 已拒绝。下方两个 VF 间以无箭头虚线和问号标出跨租户 RoCE 待验证。两张独立的绿色卡片记录各自网络的同租户测试通过。](images/fig-19-02-01-tenant-network-enforcement.png)

图 19-02-01｜跨租户的默认网 TCP 在执行点被拒绝。跨租户的附加网 RoCE 尚未测试，两张绿色卡分别记录同租户的 TCP 和 RoCE 成功对照。表中的 NAD 引用拒绝只验证附加网选择的边界，RoCE 转发还需单独验证。

本轮暂缓开放这项共享计算池能力。要满足 RoCE 隔离，先检查两个租户的实际转发域、VF 与执行点，并确认该执行点能约束目标 RDMA 流量。再观察同租户成功及跨租户按预期被拒绝的结果。跨租户试验单独超时还可能来自监听、路由或设备准备失败，需要结合有效的允许对照及执行证据。完成后保留有效规则、版本、测试条件和结果，供下一次节点或网络更换后的复验。

同租户成功对照的用途，是排除“网络或应用本来就不能工作”；执行点的规则与计数则把拒绝结果对应到隔离策略。若双向隔离是交付目标，应分别检验甲到乙和乙到甲；若同机 VF 与跨节点 VF 的转发路径不同，也要覆盖这两类路径。此处不把一份默认网策略推定为所有附加网络的通用实现。

到这里，交付目标已经包含允许谁访问、访问哪些卡、哪些通信应通过。下一篇把这些条件带入 h8 的正常分层验收：先允许受控探针走通真实生产路径，再依据完整证据开放生产。
