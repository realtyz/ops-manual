
在基于 SR-IOV 的 RoCE 接入中，每个 Pod 用一个 VF 交付网络接口和关联的 RDMA 能力。若同一批作业允许共用适配器，可以改变设备交付粒度。比较时分别问，Kubernetes 在数什么，Pod 收到哪种网络接口，RDMA 设备在哪个命名空间可见，实际数据还会争用哪些硬件资源。

## Macvlan 网络接口与共享 HCA

**HCA（Host Channel Adapter，主机通道适配器）** 在这里指向进程提供 RDMA 端口和操作能力的适配器。从一张支持 RDMA 的主机网卡 `ens1f0` 出发，Macvlan CNI 在它上面为每个 Pod 派生一个独立 MAC 的子接口。IPAM 为每个子接口分配地址。另一条交付路径由 **RDMA Shared Device Plugin** 完成，它选择该主机接口对应的 HCA，把关联的 RDMA 设备访问提供给申请共享扩展资源的容器。配置时用 `MacvlanNetwork.spec.master` 与设备插件的 `selectors.ifNames` 选中同一底层端口，实际建连时再核对 GID、地址与端口。 [NVIDIA v26.7.1 的 Macvlan 共享案例](https://docs.nvidia.com/networking/display/kubernetes2671/quick-start/macvlan-rdma-shared.html) · [RDMA Shared Device Plugin 工作流](https://github.com/Mellanox/k8s-rdma-shared-dev-plugin)

```json
{
  "resourceName": "rdma_shared_device_a",
  "resourcePrefix": "rdma",
  "rdmaHcaMax": 63,
  "selectors": {"ifNames": ["ens1f0"]}
}
```

上面是独立对照场景里的单池配置摘录，假设只选到一个 HCA，底层端口工作正常。两个 Pod 各申请 `rdma/rdma_shared_device_a: 1` 时，可各有一个 Macvlan 接口和 IP，且共同使用该 HCA。`rdmaHcaMax: 63` 给这个池提供最多 63 个可申报的逻辑单位，调度余量在本例为 `63 − 2 = 61`。实际仍只有一个物理端口，能同时接入多少 Pod 还受地址等条件限制。端口带宽由在该方向上并发的流量共用，63 个申请单位没有规定每个 Pod 能得到的带宽。 [RDMA Shared Device Plugin 的 rdmaHcaMax 定义](https://github.com/Mellanox/k8s-rdma-shared-dev-plugin#rdma-shared-device-plugin-configurations)

## VF 接口、RDMA 设备和访问权限

回到 VF 方案。一只 Pod 分到的是独立的 PCI 功能。SR-IOV CNI 把 VF 的 Linux 网络接口（netdev）移入 Pod 的网络命名空间。配置 `isRdma: true` 时，设备插件还为申请它的容器提供 RDMA 设备资源。Linux 的 RDMA 子系统另外决定进程在哪个网络命名空间能看到这个 RDMA 设备。判断某容器能否用它，需同时核对接口归属、RDMA 设备可见性、`/dev/infiniband` 的设备节点与访问权限，以及用户态 Verbs 和对应的驱动库。 [SR-IOV Device Plugin 的 isRdma](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin) · [iproute2 的 RDMA 命名空间模式](https://github.com/iproute2/iproute2/blob/main/man/man8/rdma-system.8)

主例假设节点使用 `shared`，VF 的 RDMA 设备可跨网络命名空间可见，Macvlan 共享场景也用这个模式。改用 `exclusive` 时，RDMA 设备只在一个网络命名空间内可见。为了把 VF 的 RDMA 设备交给 Pod，在 SR-IOV CNI 已接入 netdev 后，由链式 **RDMA CNI** 找到关联的 RDMA 设备并移入同一命名空间。正常删除时，链式插件按反序运行，RDMA CNI 先处理 RDMA 设备的归还。 [RDMA CNI 上游说明](https://github.com/k8snetworkplumbingwg/rdma-cni) · [CNI 插件链的 DEL 顺序](https://www.cni.dev/docs/spec/)

```yaml
# 独占变体的字段摘录，省略目标节点选择器和其余配置
# SriovNetworkPoolConfig.spec
rdmaMode: exclusive
---
# SriovNetwork.spec
metaPlugins: |
  {"type":"rdma"}
```

NVIDIA v26.7.1 用 `SriovNetworkPoolConfig` 对选中的节点设置模式，再用 `SriovNetwork.spec.metaPlugins` 生成按 `sriov`、`rdma` 排列的 NAD 插件链。模式作用于节点的 RDMA 子系统。已有活动网络命名空间且存在 RDMA 设备时，从 `shared` 切换到 `exclusive` 可能返回 `EBUSY`。先用 `rdma system show` 核对生效模式。然后在主机和 Pod 中用 `rdma link show` 按同一个 VF 核对设备可见性，最后在实际工作容器中验证 Verbs 能打开设备。 [NVIDIA v26.7.1 的 RDMA 独占模式示例](https://docs.nvidia.com/networking/display/kubernetes2671/deployment-guide-kubernetes.html#network-operator-deployment-and-rdma-exclusive-subsystem-mode) · [iproute2 的模式切换条件](https://github.com/iproute2/iproute2/blob/main/man/man8/rdma-system.8)

![两图均把 VF 0 的网络接口 net1 放在 Pod 的网络命名空间。shared 模式下主机和 Pod 都可枚举 VF 0 对应的 RDMA 设备 X。exclusive 模式下，RDMA CNI 成功接入后只有 Pod 的网络命名空间能枚举 X。](images/fig-13-05-01-rdma-netns-visibility.png)

*图 13-05-01　同一 VF 在两种独立配置中的可见性。两侧的 `net1` 都已移入 Pod。枚举结果只说明命名空间可见性，打开设备还取决于工作容器的设备授权和软件。*

## 按设备粒度与共用容量选择接入方式

| 接入方案 | 申请和接入的对象 | 可见性与共用条件 |
| --- | --- | --- |
| Macvlan + RDMA Shared Device Plugin | 一个逻辑单位和一个派生的 Macvlan 接口 | 本例用 `shared`，多只 Pod 共用 HCA 和链路。适用于允许共享适配器和性能干扰的池。 |
| SR-IOV VF | 一个 VF 和它的 netdev | 可用 `exclusive` 配合 RDMA CNI 将设备放入目标网络命名空间。同一 PF 下的 VF 仍共用物理端口。VLAN、限速和硬件隔离要按设备与策略能力验收。 |
| Host Device | 已存在的主机接口。本例对照的是一个整物理接口 | Host Device CNI 移动接口。RDMA 的设备访问还需对应插件与命名空间配置。交出整条专用接口时，节点应另有管理和默认网络路径。 |

例如，要求两个 Pod 有独立 IP 且允许共用设备时，可从共享方案起步。要求它们各有一个可追踪的 PCI 功能，并限制关联 RDMA 设备的命名空间可见性时，可选择 VF + `exclusive`。如果还要求不同的物理链路，就要让设备分别来自不同物理端口，并核对两条上联的共用部分。NVIDIA v26.7.1 的混合 Host Device 和 Macvlan 示例也将两种接入放在不同 NIC 上。 [CNI host-device 插件](https://www.cni.dev/plugins/current/main/host-device/) · [NVIDIA v26.7.1 的各类接入示例](https://docs.nvidia.com/networking/display/kubernetes2671/deployment-guide-kubernetes.html)

若将底层改成原生 **InfiniBand（IB）**，可按方案选择 IB SR-IOV CNI 或 IPoIB（IP over InfiniBand）CNI。物理口需处于 IB 模式，由子网管理器建立 fabric 的运行状态，并核对 P_Key（Partition Key，分区键）是否允许端点通信。仍然可以按设备申请、网络接入和应用使用逐层检查，具体的地址和分区条件来自 IB。 [NVIDIA v26.7.1 的 IPoIB 与 IB SR-IOV 示例](https://docs.nvidia.com/networking/display/kubernetes2671/deployment-guide-kubernetes.html)
