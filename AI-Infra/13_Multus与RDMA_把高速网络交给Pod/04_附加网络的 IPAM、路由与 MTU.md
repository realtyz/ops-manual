# 附加网络的 IPAM、路由与 MTU

硬件 VF 已经顺利移进 Pod 成为 `net1`。可一张没有 IP 地址的网卡还发不出 RoCEv2 报文。就算两端都配上了地址，如果出口选错、回程不通，或者沿途某台交换机的报文尺寸被意外截断，高速网络照样寸步难行。

本篇沿 **IP 分配 → 路由验证 → 路径 MTU 与 RoCE 参数核对** 三步，把附加网络的三层连通性彻底打牢。

## NV-IPAM 如何从全局地址池切出节点地址块

**IPAM（IP Address Management，IP 地址管理）** 负责在 CNI `ADD` 时为附加接口分配不冲突的 IP，并可同步下发网关与静态路由。在我们的 RoCE 附加网配置中，NAD 指定了 `ipam.type: nv-ipam` 与 `poolName: roce-pool`。

**NV-IPAM** 采用“控制器分大块、节点发小号”的两级设计。先在集群中声明全局 `IPPool`：

```yaml
# NV-IPAM IPPool：为命中标签的节点划定 RoCE 二层子网与每节点地址块大小
apiVersion: nv-ipam.nvidia.com/v1alpha1
kind: IPPool
metadata:
  name: roce-pool
  namespace: nvidia-network-operator
spec:
  subnet: 192.168.50.0/24           # 整个 RoCE VLAN 共享的二层网段
  perNodeBlockSize: 16              # 为每台节点切分 16 个互斥 IP 的专属地址块
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: ai-infra.example/roce-lab
            operator: In
            values: ["true"]
```

NV-IPAM 控制器监听该 `IPPool`，在 `status.allocations[]` 中为每台匹配节点划出互斥的 `startIP` 与 `endIP`（例如两台节点分别持有 `.1–.16` 与 `.17–.32`）。

Pod 启动时，本机 NV-IPAM 服务直接从本地地址块挑一个空闲 IP 交给 SR-IOV CNI 配置。Pod 删除时再随 CNI `DEL` 归还地址。

注意一个关键细节：`perNodeBlockSize: 16` 决定单节点可分多少个地址，但两台节点上的 `net1` 依然配置 `/24` 子网掩码。因为它们同处一个 RoCE VLAN，可以直接经由 ARP 解析对端 MAC 通信。若集群采用每节点独立三层子网（例如改用 NV-IPAM 的 `CIDRPool`），就必须额外在底层网络为各节点前缀配置路由与回程。

如果更换其他 IPAM 插件，务必先核对**谁来保证跨节点地址唯一性**：
- **NV-IPAM / Whereabouts**：具备集群级协调，避免跨节点撞 IP。
- **`host-local`**：仅在单节点本地保存租用状态。如果在同一二层域的两台节点上共用同一段网段，极易分配出重复 IP；沿用 `host-local` 时，必须人工给每台节点划定互斥范围。

## 为什么节点还有空闲 VF，Pod 却还是起不来？

附加网接入同时受制于**物理 VF 库存**与**节点 IP 库存**两本账。哪怕只缺其中一本，接入就会中断。

看一个典型现场：某节点上报了 8 个健康 VF，但本机地址块里实际可供租用的 IP 只有 4 个。此时已有 4 个无 GPU 请求的测试 Pod 各占了 1 个 VF 与 1 个 IP，第 5 个同类 Pod 提交进来：

| 节点库存维度 | 初始可用量 | 已消耗量 | 当前余量 |
| --- | ---: | ---: | ---: |
| 健康 VF（单位：个） | `8` | `4` | `4` |
| 可租用 IP（单位：个） | `4` | `4` | `0` |

因为调度器只认 Kubernetes 扩展资源（健康 VF 还有 4 个），第 5 个 Pod 会顺利通过调度并拿到一个 VF。可紧接着 SR-IOV CNI 调用本地 NV-IPAM 时，IP 余量为 `0`，接入直接报 `FailedCreatePodSandBox` 失败。

> [!tip] 工程直觉
> 排查 IP 耗尽时，只看 `IPPool.status.allocations[]` 的起止地址段远远不够——那只是划给节点的边界。真实余量必须核对本机 IPAM 的**实际租用记录与保留/排除地址列表**。

## 用 `ip route get` 锁定出口、源地址与默认路由陷阱

拿到双套 IP 后，进到 Pod 内用三条只读命令，就能一眼验证每个目标地址到底走 `eth0` 还是 `net1`：

```sh
ip -br addr                       # 核对 eth0 与 net1 的实际 IP 与 UP 状态
ip -4 rule show                   # 检查是否存在自定义策略路由规则
ip -4 route get 192.168.50.21     # 查对端 RoCE 地址：应返回 dev net1 src 192.168.50.11
ip -4 route get 203.0.113.10      # 查外部/集群地址：应继续走默认路由 dev eth0
```

如果应用绑定了特定源地址，可在查询时追加 `from 192.168.50.11` 复核。同时别忘了到对端 Pod 反向查询回程路由，确保去程与回程对称落在 `net1` 上。

> [!warning] 常见误区：慎给附加网络配 `default-route`
> 如果在 Pod 注解中给附加网加上 `"default-route":["192.168.50.1"]` 且网关在 `net1` 上可达，Pod 的默认出口就会从 `eth0` 切到 `net1`。原有更具体的前缀路由仍有效，但依赖默认路由的外部访问或集群服务可能立刻断连——改动后必须逐一复查 DNS、API Server 与协调服务路由。

## 从无分片 IP 探测追到 RoCE 路径 MTU

确认路由正确后，下一步要核对整条链路能否无分片通过预期大小的报文。设路径的 IPv4 MTU 为 $1500\text{ B}$，无选项 IPv4 首部占 $20\text{ B}$，ICMP Echo 首部占 $8\text{ B}$，则不分片 Ping 的最大载荷上限为：

$$\text{无分片 ICMP Echo 载荷上限} = \text{路径 IPv4 MTU} - 20\text{ B} - 8\text{ B} = 1500 - 28 = 1472\text{ B}$$

在两端 Pod 分别用 `ping -M do`（禁止分片）对比小包与满载边界包，即可快速抓出沿途 MTU 缩水问题：

```sh
ip -d link show dev net1                         # 确认本地 net1 配置的链路 MTU
ping -4 -c 3 -I net1 -M do -s 56 192.168.50.21   # 1) 小包基线：验证基本三层可达
ping -4 -c 3 -I net1 -M do -s 1472 192.168.50.21 # 2) 满载无分片探测：验证 1500 B 路径 MTU
```

如果 `56 B` 能通而 `1472 B` 丢包，应逐跳排查本端 VF、沿途交换机端口帧尺寸、回程链路与 ICMP 策略。假设路径某处实际 IPv4 MTU 只有 $1400\text{ B}$，无分片上限就会立即收缩为 $1400 - 20 - 8 = 1372\text{ B}$。

最后一步，把 IP 连通性接回 RoCE 硬件栈。无分片 Ping 只证明给定大小的 IP 报文能往返；要验证 RoCEv2 数据面，还需用 `rdma link show` 把 `net1` 对到具体 RDMA 端口，核对建连所用的 GID 条目，并对照 `ibv_devinfo` 的端口 `active_mtu` 与应用实际建立的 QP 路径 MTU。

> [!summary] 核心结论
> 附加网络不仅要“有 VF”，还要同时过三关：**NV-IPAM 节点块有空闲 IP**、**`ip route get` 确认收发走 `net1` 且未劫持默认路由**、**无分片 Ping 与 `active_mtu` / GID 核对过关**。三关齐备，才算为 RDMA 建连铺平了道路。

前四篇讲的都是“每个 Pod 独占一个 SR-IOV VF”的标准路径。如果集群里节点需要跑几十个轻量推理或开发 Pod，VF 数量不够分怎么办？或者多租户场景要求不同 Pod 互相连 RDMA 设备名都看不见，又该如何隔离？下一篇我们系统对比共享 HCA、独占 VF 与 Linux RDMA 命名空间模式。

> [!info]- 参考资料
> - **NV-IPAM、Whereabouts 与 host-local**：[NVIDIA v26.7.1 IPPool 示例](https://docs.nvidia.com/networking/display/kubernetes2671/quick-start/sriov-network-rdma.html)、[NV-IPAM 的地址分配流程](https://github.com/Mellanox/nvidia-k8s-ipam#ip-allocation-flow)、[NV-IPAM 的排除范围与分配流程](https://github.com/Mellanox/nvidia-k8s-ipam)、[Whereabouts 配置与作用范围](https://github.com/k8snetworkplumbingwg/whereabouts)、[CNI host-local 地址状态](https://www.cni.dev/plugins/current/ipam/host-local/)、[Kubernetes 扩展资源规则](https://v1-36.docs.kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources)
> - **Linux 路由、Multus 默认路由与 Ping/RFC**：[Linux ip-route 手册](https://man7.org/linux/man-pages/man8/ip-route.8.html)、[Multus 的 default-route 行为](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/how-to-use.md#specifying-a-default-route-for-a-specific-attachment)、[RFC 791 IPv4 首部](https://www.rfc-editor.org/rfc/rfc791.html)、[RFC 792 ICMP Echo 格式](https://www.rfc-editor.org/rfc/rfc792.html)、[iputils Ping 参数](https://github.com/iputils/iputils/blob/master/doc/ping.xml)
> - **RoCE GID 与 rdma-core 端口/QP 属性**：[NVIDIA RoCE 的 GID 说明](https://networking-docs.nvidia.com/doca/archive/3-5-0/rdma-over-converged-ethernet)、[rdma-core 的端口属性](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_query_port.3)、[rdma-core 的 QP 路径属性](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_modify_qp.3)
