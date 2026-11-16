
`default/rank-0` 已拿到一个 VF。为了让它通过 `net1` 与另一节点的 `rank-1` 通信，SR-IOV CNI 还要为接口配置可用地址。两个 Pod 获得地址后，仍须确认实际选择的出口、回程以及路径允许的报文尺寸。

## NV-IPAM 的地址池、节点地址块与 Pod 地址

**IPAM（IP Address Management，IP 地址管理）** 负责为接入的接口选取地址，并可返回网关与路由配置。本例的 NAD 指定 `ipam.type: nv-ipam`、`poolName: roce-pool`。NV-IPAM 控制器从选定的 `IPPool` 为符合条件的节点分配互斥地址块。节点侧服务在 CNI `ADD` 时从本节点块选出一个可用地址，再由 SR-IOV CNI 将返回结果配置到 VF 接口上。正常 `DEL` 归还地址。 [NVIDIA v26.7.1 IPPool 示例](https://docs.nvidia.com/networking/display/kubernetes2671/quick-start/sriov-network-rdma.html) · [NV-IPAM 的地址分配流程](https://github.com/Mellanox/nvidia-k8s-ipam#ip-allocation-flow)

```yaml
# 教学 IPPool，省略其他可选字段
apiVersion: nv-ipam.nvidia.com/v1alpha1
kind: IPPool
metadata:
  name: roce-pool
  namespace: nvidia-network-operator
spec:
  subnet: 192.168.50.0/24
  perNodeBlockSize: 16
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: ai-infra.example/roce-lab
            operator: In
            values: ["true"]
```

例如在一份教学快照中，`IPPool.status.allocations[]` 的 `nodeName`、`startIP`、`endIP` 显示 `gpu-h0` 持有 `.1–.16`，`gpu-h1` 持有 `.17–.32`。主例的 `.11`、`.21` 都落在各自节点的块内。地址是一次分配结果，实际落点按线上状态核对。这里的 16 是一块含有多少地址，两张接口仍配置 `/24` 掩码。它们处在同一 VLAN，可以直接解析对端的链路层地址。若改为每节点一个独立子网，例如使用 NV-IPAM 的 `CIDRPool`，还需给各节点前缀安排路由与回程。 [NV-IPAM 的 IPPool、CIDRPool 和状态字段](https://github.com/Mellanox/nvidia-k8s-ipam#ip-allocation-flow)

换 IPAM 时，先确认谁负责地址唯一性。Whereabouts 在多节点之间协调分配。`host-local` 把分配状态保存在本节点，若在同一二层域的两台节点上直接复用相同地址范围，可能发出重复地址。保留 `host-local` 时，可以为各节点设互斥的范围。 [Whereabouts 配置与作用范围](https://github.com/k8snetworkplumbingwg/whereabouts) · [CNI host-local 地址状态](https://www.cni.dev/plugins/current/ipam/host-local/)

## VF 与地址分别限制接入容量

另取一个独立快照。某节点发布 8 个健康 VF，已分配地址块中实际可租的地址有 4 个。当前有 4 个测试 Pod 各申请 1 个 VF，并各租用 1 个地址，这些 Pod 均无 GPU 请求。第 5 个测试 Pod 同样只申请 1 个 VF。

| 本节点的库存 | 初始可用 | 已消耗 | 余量 |
| --- | ---: | ---: | ---: |
| 健康 VF，单位为个 | 8 | 4 | 4 |
| 可租用 IP，单位为个 | 4 | 4 | 0 |

其他条件满足时，第 5 个 Pod 通过节点资源筛选并能分到 VF。CNI 调用本地 IPAM 时，剩余地址却是 0，接入无法完成。如果只释放一个已占地址并由 IPAM 确认可重用，新 Pod 便有机会完成该步骤。判断剩余地址要查本节点的租用和保留记录，块的起止范围只能说明分给节点的地址边界。 [Kubernetes 扩展资源规则](https://v1-36.docs.kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources) · [NV-IPAM 的排除范围与分配流程](https://github.com/Mellanox/nvidia-k8s-ipam)

## 用对端地址检查出口和回程

回到主例，从 `rank-0` 查询附加地址 `192.168.50.21` 和假定只有默认路由可达的目标 `203.0.113.10`。查询时观察返回的 `dev` 和 `src`。实际应用绑定了源地址时，可在查询中带上 `from 192.168.50.11`，并结合 `ip rule` 检查对应的策略路由。 [Linux ip-route 手册](https://man7.org/linux/man-pages/man8/ip-route.8.html)

```sh
ip -br addr
ip -4 rule show
ip -4 route get 192.168.50.21
ip -4 route get 203.0.113.10
```

在本例的单路由表假设下，`.21` 命中 `192.168.50.0/24`，查询返回 `dev net1 src 192.168.50.11`。到另一端反向查 `.11`，对应源地址应为 `.21`。对 `203.0.113.10` 的查询才使用 `eth0` 默认路由。若在另一份配置中给附加网络设置 `"default-route":["192.168.50.1"]`，并假定 `.1` 在 `net1` 上可达，原先依赖默认路由的目标可能改走 `net1`。原有更具体的路由仍参与查找，所以要用 DNS、API 或协调服务的实际目标逐一复查。这个同子网的双端例依靠直连路由和邻居解析。 [Multus 的 default-route 行为](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/how-to-use.md#specifying-a-default-route-for-a-specific-attachment)

## 从无分片 IP 探测到 RDMA 路径 MTU

**MTU（Maximum Transmission Unit，最大传输单元）** 约束可通过的报文尺寸。设整条路径的 IPv4 MTU 为 `1500 B`，IPv4 首部无选项，占 `20 B`，ICMP Echo 首部占 `8 B`。可发送的无分片 Echo 数据上限为 `1500 − 20 − 8 = 1472 B`。以下命令用 iputils Ping 比较小包与边界包，分别在两端运行并替换对端地址。 [RFC 791 IPv4 首部](https://www.rfc-editor.org/rfc/rfc791.html) · [RFC 792 ICMP Echo 格式](https://www.rfc-editor.org/rfc/rfc792.html) · [iputils Ping 参数](https://github.com/iputils/iputils/blob/master/doc/ping.xml)

```sh
ip -d link show dev net1
ping -4 -c 3 -I net1 -M do -s 56 192.168.50.21
ping -4 -c 3 -I net1 -M do -s 1472 192.168.50.21
```

若小包通而边界包失败，检查本端 VF 接口、沿途端口与帧尺寸、回程和 ICMP 策略。在其余条件相同且路径 IP MTU 确定只有 `1400 B` 的变体中，上限变为 `1372 B`。对 RoCE 还需用 `rdma link show` 关联 `net1` 与 RDMA 设备端口，核对本次建连使用的 GID 条目，并比较 `ibv_devinfo` 的端口 `active_mtu` 与应用的 QP（Queue Pair，队列对）路径 MTU。IP 探测验证给定尺寸的 IP 往返，RDMA 路径由指定设备和参数的实际传输验证。 [NVIDIA RoCE 的 GID 说明](https://networking-docs.nvidia.com/doca/archive/3-5-0/rdma-over-converged-ethernet) · [rdma-core 的端口属性](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_query_port.3) · [rdma-core 的 QP 路径属性](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_modify_qp.3)
