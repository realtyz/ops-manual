# RDMA 接入故障与网络策略覆盖

生产集群里常遇到两类棘手工单：第一类是“Pod 已经能解析集群 DNS，可高速附加网 `net1` 就是起不来或跑不通 RDMA”；第二类是安全团队的质问——“明明已经配了严格的 `NetworkPolicy` 拒绝跨租户访问，为什么测试 Pod 还能顺着 `net1` 直接连到别的租户？”

本篇把这两类问题彻底拆开讲透：前半篇结合典型命令与输出沉淀**五层接入排障法**，后半篇厘清**附加网络的多租户控制面准入与数据面策略边界**。

## 第一步：先锁死同一次接入的身份锚点

排查多网络故障最忌把“重建前的旧日志”和“重建后的新 Pod”混着读。一旦 Pod 被控制器重建，它的 UID、分配到的 VF BDF、租用的 IP 乃至落点节点都可能改变。

动手前，先为双端 Pod 锁定同一组身份锚点：**命名空间、Pod UID、所在节点、采集时间、资源键（`nvidia.com/sriov_resource`）、NAD（`default/roce-vf`）与接口名（`net1`）**。随后沿着控制面到容器内的顺序依次取证。

## 四组核心取证命令与典型输出解读

下面按“从外部事件到容器内部”的顺序，给出最常用的四类核查视图。

### 1. Pod 事件与 `network-status` 注解（排查调度、设备交接与 IPAM）

```bash
# 查看 Pod 调度结果、失败事件与 Multus 回写的 network-status 注解
kubectl describe pod -n default rank-0
kubectl get pod -n default rank-0 -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}'
```

```text
# --- 典型故障片段 A：VF 已交接，但 NV-IPAM 节点块地址耗尽 ---
Events:
  Type     Reason                  Message
  ----     ------                  -------
  Normal   Scheduled               Successfully assigned default/rank-0 to gpu-h0
  Warning  FailedCreatePodSandBox  Failed to create pod sandbox: ...
           [default/rank-0/roce-vf]: error adding container to network "roce-vf":
           nv-ipam: no available IP addresses in pool "roce-pool"

# --- 典型故障片段 B：NAD 注解资源键与 Pod 申请资源键不一致 ---
  Warning  FailedCreatePodSandBox  ... failed to get deviceID for resource "nvidia.com/sriov-other"

# --- 接入成功时的 network-status 注解输出 ---
[
  {"name":"cluster-default","interface":"eth0","ips":["10.244.0.10"],"default":true},
  {"name":"default/roce-vf","interface":"net1","ips":["192.168.50.11"]}
]
```

### 2. 容器内接口与 RDMA 设备视图（确认 `net1` 映射与链路就绪）

```bash
# 进入已启动 Pod，核对双接口 IP、RDMA 设备及端口链路状态
kubectl exec -n default rank-0 -- ip -br addr
kubectl exec -n default rank-0 -- rdma dev
kubectl exec -n default rank-0 -- rdma link show
```

```text
# ip -br addr 典型输出（eth0 为默认网，net1 为 RoCE 附加网）
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0@if18        UP             10.244.0.10/24
net1             UP             192.168.50.11/24

# rdma dev / rdma link show 典型输出（确认 mlx5_0 端口 1 为 ACTIVE 且绑定 net1）
link mlx5_0/1 state ACTIVE physical_state LINK_UP netdev net1
```

## 五层递进排障矩阵：找准最早断掉的那一环

对照上面采到的证据，按下表从上到下定位**第一个尚未完成的交接层**，即可避免盲目猜测：

| 现场观测现象 | 已确认过关的边界 | 紧接着必须核查的证据 |
| --- | --- | --- |
| `spec.nodeName` 为空，`FailedScheduling` 报 VF 不足 | 尚未选中能满足数量请求的节点 | 核对 Pod 完整资源键、各候选节点 `Allocatable` 与已绑定请求，复算单节点 VF 余量 |
| Pod 已绑定节点，但 kubelet 报拓扑准入失败 | 调度器数量检查通过，节点内缺共同 NUMA 亲和 | 核对 kubelet Topology Manager 策略、作用域，以及 GPU 与 VF 上报的 NUMA 提示 |
| 已有本次 VF 分配记录，事件报 `FailedCreatePodSandBox` | kubelet 已分配 VF BDF，sandbox 建网中断 | 若报错指向设备，比对 NAD 资源键与 Multus 传入的 `deviceID`；若指向 IPAM，查节点地址块租用与保留表 |
| `net1` 有 IP 且指定接口 Ping 通，主机内存 RDMA 失败 | 附加网三层 IP 往返已通 | 用 `rdma link show` 与 `ibv_devinfo` 核对端口、GID 与 `net1` 映射，区分设备打开、注册还是建连失败 |
| 主机内存 RDMA 跑通，换用 GPU 显存后注册失败 | 同端口主机内存 RDMA 已验证可用 | 在报错端核对 CUDA 设备号、构建参数、DMA-BUF / `nvidia-peermem` 模块及 PCIe 路径 |

看一个实战对照。若事件同时出现 `FailedCreatePodSandBox` 与地址池耗尽，且 PodResources 里的 `0000:41:00.2` 与 Multus 传给 SR-IOV CNI 的 `deviceID` 一致，就能直接跳过设备层去查本地 IPAM 租用表。

反之，若 NAD 的资源键和 PodResources 的键对不上，必须先修好设备交接。问题修复后，务必在新一次接入中同时核验 `network-status`、`ip -br addr` 与两端 RDMA 传输。

> [!tip] 工程直觉：Multus 日志在哪找？
> 不同部署形态（thin 插件模式 vs thick 守护进程模式）和日志级别配置下，Multus 的详细报错落点不同。遇到 `FailedCreatePodSandBox` 外层信息不足时，直接核对节点上的 Multus 配置（如 `logFile` / `logLevel`）和容器运行时日志。

## 为什么默认 `NetworkPolicy` 管不到 `net1`？

搞定连通性后，多租户训练与推理混部集群紧接着面临安全隔离问题。这里必须把**控制面准入**和**数据面过滤**彻底拆开，二者互不替代：

1. **控制面准入（Multus 命名空间隔离）**：Multus 的 `namespaceIsolation` 与 `globalNamespaces` 参数在 **Pod 解析网络注解时生效**，只用来限制“哪些命名空间的 Pod 有权引用哪张 NAD”。它管的是**能不能接上这张网**，不管接上之后能不能互通。
2. **数据面过滤为何失效**：标准 Kubernetes `NetworkPolicy` 由默认网络 CNI（如 Cilium、Calico）在宿主机虚拟网卡路径（veth / eBPF / iptables）上执行规则。而 SR-IOV VF 直接绕开宿主机虚拟网桥，数据包由网卡硬件直发交换机——**默认 CNI 的钩子根本看不见 `net1` 上的流量**。

要给附加网络施加数据面访问控制，必须引入面向附加网的策略对象 **`MultiNetworkPolicy`** 与真正落在数据路径上的执行组件。

## `MultiNetworkPolicy` 与网卡 representor 的硬件卸载条件

在硬件加速网络栈里，`multi-networkpolicy-tc` 是实现 `MultiNetworkPolicy` API 的典型执行组件。它的落地依赖三项严格的硬件与架构前提：

- **网卡运行在 switchdev 模式**：网卡内部嵌入硬件交换平面，支持 Linux **TC（Traffic Control）硬件卸载**。
- **存在 VF 对应的 representor 接口**：**representor** 是宿主机侧代表该 VF 在网卡内部交换机上端口的 Linux 网络接口。`multi-networkpolicy-tc` 监听 `MultiNetworkPolicy` 变化，把允许/拒绝规则下发到对应 Pod VF 的 representor 上，由网卡硬件线速过滤。
- **匹配受支持的附加 CNI**：上游 `multi-networkpolicy-tc` 的支持范围是 **`accelerated-bridge`** 架构，并不直接覆盖本文主线采用的普通 `sriov` CNI。

下面是一份精简的 `MultiNetworkPolicy` 示例，展示如何仅放行同组业务 Pod 访问本附加网的特定端口：

```yaml
apiVersion: k8s.cni.cncf.io/v1beta1
kind: MultiNetworkPolicy
metadata:
  name: allow-same-job-net1
  namespace: default
  annotations:
    # 明确声明这条策略作用于哪张附加网络 NAD（而非默认网 eth0）
    k8s.v1.cni.cncf.io/policy-for: default/roce-accel
spec:
  podSelector:
    matchLabels:
      app: pd-decode-pool          # 策略保护的目标 Pod 集合
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: pd-prefill-pool # 仅允许同租户 Prefill 池通过附加网访问
```

> [!warning] 常见误区：不要把默认网策略或 NAD 隔离误当成 RoCE 数据面隔离
> 普通 SR-IOV CNI 上即使配置了 `NetworkPolicy` 和 Multus `namespaceIsolation`，同处一个 RoCE VLAN 内的已接入 Pod 在数据面上依然可能互通。确认附加网策略真正生效，必须核对规则下发到了正确的 VF **representor**，且目标协议在卸载路径上受该规则约束。

## 双网络允许与拒绝的对照验收法

验收多网络策略覆盖时，千万不要只测一条“被拒”就草率签字——连路由不通都会表现为连接失败。规范做法是固定同一对测试 Pod 与服务端监听，在**默认 IP（`eth0`）**和**附加 IP（`net1`）**上分别执行“允许对照 + 拒绝验证”：

| 测试路径 | 允许对照（策略放行） | 拒绝验证（策略封禁） | 可得出的严格结论 |
| --- | --- | --- | --- |
| **默认网 IP（`eth0` TCP）** | 连接成功（证明监听与路由有效） | 连接被拒 | 默认 CNI 的 `NetworkPolicy` 对默认网 TCP 生效 |
| **附加网 IP（`net1` TCP）** | 连接成功 | 仍成功 / 被拒 | 单独证明附加网数据面对 TCP 的放行或拦截状态 |
| **附加网端口（`net1` RoCE）** | `perftest` 成功 | 需核对策略实现是否覆盖 RoCE 卸载路径并用双端 RDMA 实测 | 证明硬件 RoCE 流量本身是否真正受到策略约束 |

> [!summary] 核心结论
> 高速网络排障要**先锁同一次接入的身份，再沿五层证据找最早断口**；多租户安全则要**把控制面 NAD 引用隔离与数据面报文过滤彻底分开**，绝不能用 `eth0` 的 `NetworkPolicy` 替代 `net1` 的数据面隔离验收。

至此，第 13 章从双网络动机、Multus 委派、SR-IOV 切分、IPAM/MTU、共享与隔离、NUMA/PCIe 对齐，一直到双 Operator 交付、GDR 实测与安全边界的全部拼图已经完整合拢。最后一篇我们通过四组无需实机的综合实战题，检验整套知识能否在复杂现场灵活迁移。

> [!info]- 参考资料
> - **Kubernetes PodResources、sandbox 事件与网络策略**：[Kubernetes v1.36 PodResources](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)、[Kubernetes v1.36 sandbox 事件实现](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/kuberuntime/kuberuntime_manager.go)、[Kubernetes v1.36 NetworkPolicy 的实现条件](https://v1-36.docs.kubernetes.io/docs/concepts/services-networking/network-policies/)
> - **Multus 日志、隔离与设备 ID 传递**：[Multus 日志配置](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/configuration.md)、[Multus 命名空间隔离配置](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/configuration.md#namespace-isolation)、[Multus v4.2.4 的设备 ID 传递](https://github.com/k8snetworkplumbingwg/multus-cni/blob/v4.2.4/pkg/k8sclient/k8sclient.go)、[NV-IPAM 的本地地址块分配](https://github.com/Mellanox/nvidia-k8s-ipam#ip-allocation-flow)
> - **附加网络策略卸载与 representor 机制**：[multi-networkpolicy-tc 的 CNI 范围与运行机制](https://github.com/k8snetworkplumbingwg/multi-networkpolicy-tc)、[Linux 内核的网络功能 representor](https://docs.kernel.org/networking/representors.html)
