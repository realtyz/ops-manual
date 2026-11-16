
`rank-0` 和 `rank-1` 可以访问集群 DNS，但运行者报告附加网络通信异常。先固定两个 Pod 的命名空间、UID、实际节点、采集时间与网络名。在同一次接入中找出最后一个已完成的交接，再查下一步所缺的输入。Pod 重建后地址、设备和事件都可能变化，取证时要重新对齐标识。

## 按一次 Pod 接入查找最早的失败

本例的资源键是 `nvidia.com/sriov_resource`，NAD 为 `default/roce-vf`，Pod 内的接口为 `net1`。以这三个键把调度记录、PodResources 的 VF BDF、Multus 交给插件的 `deviceID` 和 Pod 内接口串起来。下表用互相独立的教学快照说明读法。消息和日志位置视组件版本、Multus 的 thin 或 thick 部署方式及日志配置而定。 [Kubernetes v1.36 PodResources](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/) · [Kubernetes v1.36 sandbox 事件实现](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/kuberuntime/kuberuntime_manager.go) · [Multus 日志配置](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/configuration.md)

| 观测 | 已确认的边界 | 紧接着查什么 |
| --- | --- | --- |
| `spec.nodeName` 为空，`PodScheduled=False`，`FailedScheduling` 指向 VF 不足 | 尚未选到能满足请求的节点。 | 对照请求的完整资源名、每个候选节点的 `Allocatable` 及已绑定请求，复算余量。 |
| Pod 已绑定，kubelet 报拓扑准入失败 | 调度器已选节点，节点内没有满足策略的亲和性。 | 对照拓扑策略、作用域，以及 GPU 和 VF 的设备提示。 |
| 有本次 VF 分配记录，`FailedCreatePodSandBox` 的内层消息指向设备或 IPAM | kubelet 已选出 VF，sandbox 的网络交付尚未完成。 | 对设备错误查 NAD 的完整资源键和 Multus 的 `deviceID`。对 IPAM 错误查传入的池名、节点块和租用状态。 |
| `net1` 有地址且指定接口的 Ping 成功，主机内存 RDMA 未跑通 | 已有该次 IP 往返证据。 | 核对端口、GID 与 `net1` 的映射，再区分设备打开、注册、建连或传输阶段的错误。 |
| 同一对端口的主机内存测试成功，改用 GPU 缓冲区后注册失败 | 指定端口的普通 RDMA 已有传输记录，新增失败在显存步骤。 | 核对失败端实际选择的 GPU、CUDA、注册方式、构建和 PCIe 路径。 |

例如事件同时写了 `FailedCreatePodSandBox` 和地址池无可用地址，且 PodResources 中的 `0000:41:00.2` 与 Multus 交给 SR-IOV CNI 的 `deviceID` 一致，可以从本节点地址块开始核对。若 NAD 的资源名和 PodResources 的键不同，则应先取得设备交接成功的证据。修复后在新一次接入中同时取得 `network-status` 和 `ip -br addr`，再确认两端端口和数据传输。 [Multus v4.2.4 的设备 ID 传递](https://github.com/k8snetworkplumbingwg/multus-cni/blob/v4.2.4/pkg/k8sclient/k8sclient.go) · [NV-IPAM 的本地地址块分配](https://github.com/Mellanox/nvidia-k8s-ipam#ip-allocation-flow)

## NAD 引用权限与数据面策略执行

Multus 的 `namespaceIsolation` 和 `globalNamespaces` 决定 Pod 能引用哪些 NAD。它们在接入请求解析时生效。若目标是限制已接入 Pod 之间的通信，还需要在对应数据路径上的执行组件。默认网络上的 Kubernetes `NetworkPolicy` 由支持该 API 的网络插件执行。附加网的覆盖要根据附加 CNI、网卡模式、硬件卸载和执行点实际判断。 [Multus 命名空间隔离配置](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/configuration.md#namespace-isolation) · [Kubernetes v1.36 NetworkPolicy 的实现条件](https://v1-36.docs.kubernetes.io/docs/concepts/services-networking/network-policies/)

以一套独立于主例的 `accelerated-bridge` 部署为例。`multi-networkpolicy-tc` 实现附加网的 `MultiNetworkPolicy` API。它需要支持 switchdev 和 TC 硬件卸载的 NIC，以及 Pod 所用 VF 对应的 **representor**，即主机侧代表该 VF 在网卡内部交换机上端口的网络接口。控制器观察策略对象，并在该 representor 上配置 Linux Traffic Control（TC）规则。确认策略覆盖要同时核对规则与正确 representor 的对应关系，以及目标流量是否受该处规则约束。上游文档的支持范围是 `accelerated-bridge`，主例的普通 SR-IOV CNI 不能据此获得同等覆盖结论。 [multi-networkpolicy-tc 的 CNI 范围与运行机制](https://github.com/k8snetworkplumbingwg/multi-networkpolicy-tc) · [Linux 内核的网络功能 representor](https://docs.kernel.org/networking/representors.html)

## 对两张网络分别验证允许和拒绝

在测试网络里，固定同一对测试 Pod 和服务端监听条件，分别使用默认 Pod IP 与附加 IP。先做预期允许的对照，确认监听和路由能支持连接，再检查预期拒绝的情形。每次记录源和目的、实际接口、协议、对应策略，条件允许时再核对执行点的规则或计数。若默认 IP 上的 TCP 被拒，附加 IP 上的同类连接却通过，这两条记录只能说明各自的 TCP 行为。若还要求限制 RoCE，应先确认实现对该协议和卸载路径的覆盖，再用指定设备的双端 RDMA 试验和相应规则证据验收。 [Kubernetes v1.36 NetworkPolicy 的行为范围](https://v1-36.docs.kubernetes.io/docs/concepts/services-networking/network-policies/) · [multi-networkpolicy-tc 的执行机制](https://github.com/k8snetworkplumbingwg/multi-networkpolicy-tc)
