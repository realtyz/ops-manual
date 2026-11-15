# 附加网络的地址、路由和 MTU 怎样验收

Pod 中已经出现 `net1` 和一个 IP，仍要检查两端是否使用同一套地址计划、流量是否从预期接口发出，以及路径能承载多大的报文。以下以同一个 RoCE VLAN 上的 `192.168.50.0/24` 为例。

## IPAM 先给节点地址块，再给 Pod 地址

附加网的 CNI 配置中，`ipam.type: nv-ipam` 指定地址管理插件，`poolName: roce-pool` 引用 `IPPool`。控制器根据 `spec.subnet`、`perNodeBlockSize` 和节点选择条件，为各节点分配互不重叠的块。节点侧 IPAM 守护进程从本节点块中为一个 CNI `ADD` 分配地址，`DEL` 时处理释放。`IPPool.status.allocations[]` 的 `nodeName`、`startIP`、`endIP` 说明节点持有哪个范围。该范围内还剩几个地址，要继续核对节点的实际分配记录和保留规则。 [NV-IPAM 的配置、状态和分配流程](https://github.com/Mellanox/nvidia-k8s-ipam)

```yaml
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

这里的地址和分块用于推演。`IPPool` 从同一 `/24` 划出两个不重叠的地址范围，两个节点上的 Pod 仍用 `/24` 掩码，可以在同一二层网直连。若计划采用每节点单独的路由子网，可看 NV-IPAM 的 `CIDRPool`，并为分配到各节点的前缀安排路由和网关。

换 IPAM 时要问谁维护已分配地址。Whereabouts 在多个节点间协调分配。`host-local` 把状态存在每台节点本地，只保证该节点上的唯一分配。多台节点在同一二层直接复用同一个 `host-local` 地址范围，可能拿到相同的地址。若仍要用 `host-local`，需要按节点规划互斥范围。 [NV-IPAM 的 IPPool 与 CIDRPool](https://github.com/Mellanox/nvidia-k8s-ipam#ip-allocation-flow) · [Whereabouts](https://github.com/k8snetworkplumbingwg/whereabouts) · [CNI host-local 的状态范围](https://www.cni.dev/plugins/current/ipam/host-local/)

再看一个独立的容量算例。假设某节点有 8 个健康 VF，地址块只有 4 个可用 IP。4 个测试 Pod 各消耗一个 VF 和一个地址，均不申请 GPU。第 5 个 Pod 请求 1 个 VF，节点还有 `8 − 4 = 4` 个 VF 调度余量，IP 余量为 `4 − 4 = 0`。因此它可以通过 VF 数量筛选，也可以拿到 VF，但 CNI 无法完成地址配置。此时应核对该节点的块、存活附件和回收记录。

## 验收路由时用进程实际访问的地址

假定 `net1` 的本地地址为 `192.168.50.11/24`，对端为 `192.168.50.21`。在本地 Pod 查询实际出口，再到对端反向查 `.11`。路由查询的结果中看 `dev` 和 `src`。下面的 `203.0.113.10` 仅用来查询默认路由，假定它没有更具体的路由。 [Linux ip-route 手册](https://man7.org/linux/man-pages/man8/ip-route.8.html)

```sh
ip -br addr
ip -4 rule show
ip -4 route get 192.168.50.21
ip -4 route get 203.0.113.10
```

本例中 `.21` 命中 `net1` 的直连子网，源地址应选 `.11`，默认出口由 `eth0` 保留。去不同子网时，还要有指向目标的路由、下一跳与回程。进程若绑定源地址，可在查询中带上 `from 192.168.50.11`，并检查相关策略路由。应用使用 RoCE 时还要核对实际 RDMA 设备和端口，路由查询只显示这一条件下的 IP 查找结果。

如果在网络选择注解的 JSON 中给 `roce-vf` 加入 `"default-route":["192.168.50.1"]`，并假设 `.1` 是 `net1` 上可达的网关，那么依赖默认路由的目的地址可能被改送到 `net1`。单独复查 DNS、API 或 Rendezvous 目的地址的有效路由，已有更具体路由的目标仍按其路由规则判断。同一子网上的 RoCE 测试仅需直连路由。 [Multus 的 `default-route` 用法与影响](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/how-to-use.md#specifying-a-default-route-for-a-specific-attachment)

## 小包通过后核对路径 MTU

`ip -d link show dev net1` 可以看 VF 的 IP 接口 MTU，主机还要核对 PF 与交换路径实际允许的报文大小。假设整个路径的 IP MTU 为 1500 B，IPv4 首部无选项，占 20 B，ICMP Echo 首部占 8 B，那么不分片的 Echo 数据上限为 `1500 − 20 − 8 = 1472 B`。可从 `.11` 所在 Pod 做小包与边界两组探测，命令使用 iputils 的 IPv4 Ping。 [IPv4 首部](https://www.rfc-editor.org/rfc/rfc791.html) · [ICMP Echo 格式](https://www.rfc-editor.org/rfc/rfc792.html) · [iputils ping 参数](https://github.com/iputils/iputils/blob/master/doc/ping.xml)

```sh
ip -d link show dev net1
ping -4 -c 3 -I net1 -M do -s 56 192.168.50.21
ping -4 -c 3 -I net1 -M do -s 1472 192.168.50.21
```

若小包通而 1472 B 失败，检查源端 MTU、沿途较小的 MTU、返回路径及 ICMP 策略。再分别核对两端 `ibv_devinfo` 的端口 `active_mtu` 和应用实际使用的 QP 路径 MTU。RDMA 端口属性、QP 的配置与前面的 IP 载荷计算是不同的观测量。要确认 RDMA 可用，需在指定端口做实际的 RDMA 传输。 [rdma-core 端口查询](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_query_port.3) · [rdma-core QP 路径属性](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_modify_qp.3)
