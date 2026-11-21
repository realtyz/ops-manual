# GPU 管理权限、设备访问与租户隔离

集群里明明给租户甲设了“最多 4 张 GPU”的配额，也配好了 Kubernetes `NetworkPolicy` 阻断跨命名空间流量，为什么租户甲的测试 Pod 依然有可能窥见宿主机上的全部 8 张卡，甚至通过高速网卡把报文直接打进租户乙的训练任务里？

原因在于：**GPU 集群的隔离不是单一开关，而是一条跨越 API 控制面、容器运行时与硬件高速网络的多层防线。** 漏掉任何一层，其他层的安全承诺都会瞬间失效。

在开放新节点接入多租户资源池之前，我们先在已验收节点上用两个测试租户（租户甲、租户乙）的小规模探针把所有隔离边界跑通；待交付节点入池时，将原封不动继承这套已验证的规则。

## 一、一次租户请求经过的四道权限边界

把“租户隔离”拆成可观测的工程事实，只需盯住一条请求从提交到收发数据所经过的四道关卡：

1. **API 授权（RBAC）**：判断该身份能对哪些对象做什么操作（能否建 Pod、能否篡改平台组件或节点标签）。
2. **Pod 准入（PSA 与策略校验）**：哪怕有权建 Pod，也要审查模板字段是否夹带特权模式、宿主机路径挂载、非法 `RuntimeClass` 或验收污点容忍。
3. **设备交付与运行时边界**：确认容器实际拿到手的物理 GPU（或 MIG 实例）与 kubelet 分配清单完全一致，不存在绕过分配器的直通后门。
4. **双网数据面执行点**：确认接口创建完成后，默认网络与 RoCE 附加网络都在真实转发路径上拦截了越权报文。

| 防线层级 | 解决的核心安全问题 | 验收时必须拿出的成对证据 |
| --- | --- | --- |
| **API 授权** | 能否修改 `ClusterPolicy`、节点标签或管理组件 | 租户身份提交业务对象允许，越权修改平台对象被拒绝 |
| **Pod 准入** | 模板能否索要 `privileged`、`hostPath` 或管理运行时 | 合规模板放行，违规模板被准入控制器拦截并记录原因 |
| **设备交付** | 容器内可见的卡是否严格等于本次分配结果 | 对齐 Pod UID、kubelet PodResources 清单与容器内 GPU UUID |
| **双网执行** | 网卡已就绪后，真实流量到底能不能穿透到另一租户 | 分别在默认网与 RoCE 附加网完成“同租户通 + 跨租户拒 + 命中计数”对照 |

> [!summary] 核心结论
> 控制面拒绝发生在对象写入集群之前，数据面拒绝发生在网络报文转发路径之上。**两者各自护住不同层次，绝不能用控制面的拒绝记录替代数据面的真实隔离验证。**

## 二、管理组件特权、RBAC 与 Pod 安全准入

上一篇确定了节点软件由主机镜像与 GPU Operator 分别管辖。为了配置容器运行时并探测宿主机设备，GPU Operator 下发的 Toolkit、Device Plugin 等管理容器不可避免地需要 `hostPID`、宿主机目录挂载或特定 `securityContext` 权限。

因此，平台必须把管理组件与租户负载彻底拆开治理：

- **管理命名空间（如 `gpu-operator`）**：在 Pod Security Admission（**PSA**）中显式放行 `privileged` 级别，但通过 **RBAC** 将该命名空间的写权限严格锁死给运维自动化账号与平台管理员，同时保护关联的 `ServiceAccount` 与权限绑定不被低权角色接管。
- **租户命名空间**：用 RBAC 限定租户仅能管理本命名空间内的常规工作负载，**禁止修改命名空间标签、节点污点或任何集群级资源**；同时在命名空间打上受限的 PSA 标签与 GPU 扩展资源配额（`ResourceQuota`），从入口拦截特权提权与超额占卡。

下面是一套面向普通租户命名空间的最小权限与配额基线配置：

```yaml
# 1. 租户命名空间：强制启用受限 PSA，拒绝 privileged 与 hostPath 挂载
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-alpha
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.36
    pod-security.kubernetes.io/warn: restricted
---
# 2. 扩展资源配额：限制该命名空间最多声明 4 张整卡与 1 个 RDMA VF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-quota
  namespace: tenant-alpha
spec:
  hard:
    requests.nvidia.com/gpu: "4"       # 约束通过标准分配路径申请的 GPU 总额
    requests.example.com/rdma-vf: "1"  # 同步约束附加高速网卡配额
---
# 3. 最小权限 Role：允许提交工作负载，禁止操作安全标签与平台管理对象
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-workload-editor
  namespace: tenant-alpha
rules:
  - apiGroups: ["", "apps", "batch", "batch.volcano.sh"]
    resources: ["pods", "deployments", "jobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods/log", "events"]
    verbs: ["get", "list", "watch"]
```

配置生效后，不仅要验证租户可以正常拉起合规 Pod，还要用 `kubectl auth can-i` 反向实测越权路径（例如修改 `ClusterPolicy`、给节点去污点、给自己的命名空间降级 PSA 标签），确保全部返回 `no`。

## 三、堵住运行时后门：限制 `RuntimeClass` 与设备直通

仅仅配置了 `ResourceQuota` 和 PSA，仍然存在一条极易被忽略的绕过链路。

在 GPU Operator `v26.7.2` 的原生 CDI 模式（NRI 关闭）下，普通业务 Pod 申请 `nvidia.com/gpu` 后，由 Device Plugin 与 kubelet 选卡并通过 CDI 将指定设备注入容器。

但平台内部的部分管理容器为了直接盘点宿主机全量卡，会声明 `runtimeClassName: nvidia` 并配合 `NVIDIA_VISIBLE_DEVICES` 环境变量直通设备。

**关键漏洞在于：Kubernetes 标准 PSA 安全级别并不校验 `spec.runtimeClassName`。** 如果租户准入策略没有封死这条管理运行时通道，租户就可以不申请 `nvidia.com/gpu`（完美绕过 `ResourceQuota` 记账！），直接靠指定管理运行时访问宿主机上的未分配 GPU。

在 Kubernetes `v1.36` 中，可以直接用原生 `ValidatingAdmissionPolicy` 精准封堵这一越权路径：

```yaml
# 禁止租户命名空间使用平台专属的 nvidia 管理运行时或直通环境变量
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: deny-gpu-management-runtime-bypass
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: "!(has(object.spec.runtimeClassName) && object.spec.runtimeClassName == 'nvidia')"
      message: "租户 Pod 禁止使用管理类 runtimeClassName: nvidia，请通过标准资源请求获取 GPU"
    - expression: "object.spec.containers.all(c, !has(c.env) || c.env.all(e, e.name != 'NVIDIA_VISIBLE_DEVICES'))"
      message: "租户 Pod 禁止显式设置 NVIDIA_VISIBLE_DEVICES 绕过设备分配"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: bind-deny-gpu-bypass-to-tenants
spec:
  policyName: deny-gpu-management-runtime-bypass
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels:
        infra.example.com/pool-role: tenant   # 仅作用于租户命名空间，不影响 gpu-operator 管理空间
```

我们可以用两个条件变体看清各层的独立职责：把租户配额从 4 卡降到 2 卡，拦住的是“合法新请求的数量”；而放开管理 `RuntimeClass`，改变的是“运行时实际授予容器的硬件视野”——前者永远补救不了后者。

> [!warning] 常见误区：混为一谈的“设备隔离”与“故障域隔离”
> 即使两租户在一台 8 卡节点上分别拿到不同的物理整卡（或推理池里的不同 MIG 分区），它们依然共享同一套 Linux 内核、GPU 驱动、PCIe 根复合体与整机供电。
>
> 如果业务要求**租户间硬件与驱动级强故障隔离**，不能只依赖单卡分配或 MIG，必须通过节点污点/反亲和把不同租户切分到物理隔离的节点池；若退回到第 14 章的 Time-Slicing 共享，还需额外评估显存争抢与同卡故障连坐边界。

## 四、默认网络与 RoCE 附加网络的隔离执行点

进入网络层后，多租户隔离最容易踩的坑是：**以为给默认网配好了 `NetworkPolicy`，高速计算网也就跟着安全了。**

回顾第 13 章的双网架构：Pod 的默认接口承载控制面与常规服务流量，受默认 CNI 插件的 `NetworkPolicy` 管辖；而承载 RDMA/RoCE 的附加接口通过 **Multus** 委派给 SR-IOV 插件，将独立的网卡 **VF** 直接透传进容器，报文全程绕开宿主机内核协议栈。

这意味着附加高速网必须分两步设防：

1. **控制面选网权（能否接上这张网）**：在 Multus `v4.2.4` 中启用 `namespaceIsolation`，限制 Pod 只能引用本命名空间内的 `NetworkAttachmentDefinition`（NAD），同时审慎核查全局放行白名单，防止租户甲偷挂租户乙的 NAD。
2. **数据面转发隔离（接上之后能和谁通）**：虽然每个租户拿到独立的 VF，但同一节点上的多个 VF 仍共享同一物理端口（PF）和接入交换机。真正阻断跨租户 RoCE 报文，必须依赖**覆盖硬件转发路径的真实执行点**——例如在 SR-IOV NAD 中为不同租户分配隔离的 VLAN 与交换机侧 VRF/ACL，并在硬件交换机或网卡 eSwitch 层面完成策略下发。

下面是一份通过独立 VLAN 将租户甲的 SR-IOV VF 锁定在专属二层/VRF 隔离域的 NAD 片段，以及控制面与数据面联合验收命令：

```yaml
# tenant-alpha 命名空间专属的 RoCE 附加网络定义（NAD）
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: roce-tenant-alpha
  namespace: tenant-alpha
  annotations:
    k8s.v1.cni.cncf.io/resourceName: example.com/rdma-vf
spec:
  config: |
    {
      "cniVersion": "1.0.0",
      "type": "sriov",
      "vlan": 2101,
      "spoofchk": "on",
      "trust": "off",
      "ipam": { "type": "whereabouts", "range": "10.210.1.0/24" }
    }
```

```bash
# 1. 控制面权限核验：租户身份不能篡改平台对象，也不能跨空间引用 NAD
$ kubectl auth can-i patch clusterpolicy --as=system:serviceaccount:tenant-alpha:default
no
$ kubectl apply -n tenant-alpha -f probe-using-beta-nad.yaml
Error from server: ... Multus namespace isolation denied reference to tenant-beta/roce-tenant-beta

# 2. 数据面同租户允许对照（验证链路、VF 与 RDMA 栈本身健康）
$ kubectl exec -n tenant-alpha probe-a1 -- ib_write_bw -d mlx5_0 -x 3 --report_gbits 10.210.1.12
# 预期：建连成功，单向载荷速率符合正常基线

# 3. 数据面跨租户拒绝验证（须同时核查交换机/防火墙执行点命中计数！）
$ kubectl exec -n tenant-alpha probe-a1 -- ib_write_bw -d mlx5_0 -x 3 10.210.2.12
# 预期：建连被隔离域阻断，且在目标 VRF/ACL 执行点看到对应丢包计数递增
```

### 成对证据：为什么“跨租户超时”不等于“隔离通过”

下表记录了开通第二租户前的一次典型隔离验收结果。我们在两租户间固定相同的源/目的接口、地址、协议与接收端监听条件，逐项执行“同租户允许”与“跨租户拒绝”对照：

| 验证路径 | 期望允许（同租户对照） | 期望拒绝（跨租户试验） | 本轮验收裁定 |
| --- | --- | --- | --- |
| **默认网 TCP** | 连接成功，业务端口正常响应 | 被默认 CNI 策略执行点拦截，命中拒绝规则计数 | **通过**（覆盖指定端点） |
| **引用他租户 NAD** | 本命名空间 NAD 正常附加 | 引用租户乙 NAD 被 Multus 准入校验拒绝 | **通过**（选网权边界闭环） |
| **附加网 RoCE** | 租户内双端 VF RDMA 传输通过 | 仅测试了附加网 TCP 拦截规则，双端 RoCE 实测尚未执行 | **未通过**（暂缓开放多租户 RoCE） |

![租户甲与租户乙的测试 Pod 各自挂载默认接口与附加接口。上方默认网 TCP 测试箭头被策略执行点拦截，标明跨租户 TCP 已拒绝，下方绿卡记录同租户 TCP 通过。下方附加网 RoCE 路径在两个 VF 之间以虚线和问号标出跨租户 RoCE 待验证，底部绿卡记录同租户 RoCE 已通过。](images/fig-19-02-01-tenant-network-enforcement.png)

*图 19-02-01：默认网与附加网的隔离验收证据对比。跨租户默认网 TCP 已在执行点确认拒绝，而附加网仅验证了同租户连通性与 NAD 引用边界，跨租户 RoCE 转发隔离仍缺实测拒绝对照，故当前暂缓开放多租户共享池。*

面对表中的缺口，正确决策只有一个：**暂缓开放共享高速计算池，直至补齐真实的跨租户 RoCE 拒绝证据。** 这里有三条硬性判读原则：

1. **必须有同租户成功对照**：如果只测了跨租户超时，却没证明同租户能通，超时完全可能源于对端没起监听、路由写错或设备异常，而非隔离生效。
2. **必须核对执行点计数或日志**：拒绝现象要能对应到具体的 VLAN/VRF 隔离规则或 ACL 丢包计数，证明报文是在预期位置被拦下的。
3. **覆盖方向与同机/跨机路径**：若要求双向隔离，须分别验证“甲→乙”与“乙→甲”；若同一物理网卡下两个 VF 的本机硬件环回（hairpin）与跨节点经交换机转发走不同执行点，必须两条路径都测到。

> [!tip] 多租户放行前权限与隔离核对清单
> - **RBAC 与 PSA**：管理命名空间写权限仅限平台管理员；租户空间强制受限 PSA，禁止 `privileged` 与 `hostPath`。
> - **设备防绕过**：配齐 `ResourceQuota` 与 `ValidatingAdmissionPolicy`，封堵 `runtimeClassName: nvidia` 与 `NVIDIA_VISIBLE_DEVICES` 直通后门，并按 UUID 核对分配一致性。
> - **双网成对证据**：默认网 TCP、Multus NAD 隔离、附加网 RoCE 均集齐“同租户通 + 跨租户拒 + 执行点命中”记录，且覆盖同机与跨机 VF 转发路径。

到这一步，节点“装什么”和“谁能用”都已落到可执行的策略文件上。下一篇，我们把待交付的 8 卡节点放进受控验收隔离区，走通从全机硬件健康到**训练池与推理池差异化性能基线**的完整生产准入流程。

> [!info]- 参考资料
> - **GPU Operator 安全模型与模板**：[NVIDIA · GPU Operator v26.7.2 安全说明](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/security.html)、[Toolkit DaemonSet v26.7.2 权限模板](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/assets/state-container-toolkit/0500_daemonset.yaml)、[GPU Operator 的 PSA 部署前提](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/getting-started.html#prerequisites)
> - **Kubernetes 授权、准入与配额**：[Kubernetes v1.36 · RBAC 最佳实践](https://v1-36.docs.kubernetes.io/docs/concepts/security/rbac-good-practices/)、[Kubernetes v1.36 · Pod 安全标准（PSA）](https://v1-36.docs.kubernetes.io/docs/concepts/security/pod-security-standards/)、[Kubernetes v1.36 · 扩展资源配额](https://v1-36.docs.kubernetes.io/docs/concepts/policy/resource-quotas/#quota-for-extended-resources)、[Kubernetes v1.36 · Device Plugin 分配与监控](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
> - **CDI 管理运行时与 GPU 共享边界**：[NVIDIA · CDI 下的 GPU 管理容器](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/cdi.html#cdi-and-gpu-management-containers)、[NVIDIA · GPU 共享隔离边界说明](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html)
> - **默认网与 Multus 附加网隔离**：[Kubernetes v1.36 · NetworkPolicy 作用范围](https://v1-36.docs.kubernetes.io/docs/concepts/services-networking/network-policies/)、[Multus v4.2.4 · NAD 命名空间隔离配置](https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/v4.2.4/docs/configuration.md)
