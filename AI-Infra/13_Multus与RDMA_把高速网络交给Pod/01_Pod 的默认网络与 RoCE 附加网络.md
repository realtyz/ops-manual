# Pod 的默认网络与 RoCE 附加网络

两台 GPU 服务器上各自跑起一个分布式工作负载 Pod。它们已经顺着默认网络找到集群 DNS、完成了 Rendezvous 会合。接下来，训练作业要开始同步几十 GB 梯度，或者 PD 分离推理中的 Prefill 实例要把 KV Cache 搬给 Decode 实例。

这时候一个核心矛盾立刻暴露出来：**为什么不能直接拿 Kubernetes 默认的 `eth0` 去跑 200 Gb/s 甚至 400 Gb/s 的 GPUDirect RDMA？**

原因在数据路径上。默认网络 `eth0` 通常挂在宿主机虚拟网对（veth）、虚拟网桥或 Overlay 封装之上，所有报文都要经过 Linux 内核协议栈与多次内存拷贝。这套设计对 Service、DNS 和健康检查极好，却彻底堵死了网卡绕过 CPU 直读显存的硬件旁路。

工程上的标准解法非常直观：**保留 `eth0` 负责集群控制与服务发现，再给每个需要的 Pod 接上第二张“直通高速网口”（`net1`）**。本篇从 CNI 与 Multus 切入，把双网络的分工与流量判读规则立住。

## 默认网络与附加网络的分工：CNI 与 Multus

每个 Pod 拥有独立的 Linux 网络命名空间，包含专属的网络接口、IP 地址与路由表，Pod 内的全部容器共享这一视图。

Pod 启动时，容器运行时遵循 **CNI（Container Network Interface，容器网络接口）** 规范调用网络插件。默认 CNI 插件负责为 Pod 建立主接口（通常为 `eth0`），打通集群 Service、CoreDNS、监控抓取与控制面协调。

然而，标准 Kubernetes 网络模型默认只对接一个 CNI 插件。要让同一个 Pod 同时接入多张网络，就需要引入元网络插件 **Multus**：

- **什么是 Multus**：Multus 本身并不直接搬运数据包，它像一位“多网卡总调度”。部署后，容器运行时把 Pod 的建网请求交给 Multus。
- **它如何干活**：Multus 先调用集群原有的默认 CNI 建好 `eth0`，再按 Pod 的声明调用一个或多个附加网络插件（如 SR-IOV CNI），把专用高速网口作为 `net1` 接进同一个网络命名空间。

这条附加网络承载的是 RoCEv2 高速流量。因为 RoCEv2 将 RDMA 报文封装在 UDP/IP 之上，Pod 要真正跑通这条通道，必须同时满足两个条件：**拥有附加网络的 IP 地址与路由可达性**，以及**进程能打开与该接口对应的 RDMA 设备和端口**。

> [!tip] 工程直觉：两条网线服务两类流量
> 把 Pod 想象成插着两根网线的服务器。`eth0` 是办公控制网，管 DNS、API Server、训练 Rendezvous 和推理网关的健康探针；`net1` 是专用高速数据网，专供训练集合通信（AllReduce/All-to-All）和 PD 分离推理的跨 Pod KV Cache 迁移。

## 双节点 Pod 的双接口映射与物理路径

沿用第 12 章的两台双卡节点，我们在每台机器上放置一个代表性工作负载 Pod。每台节点同时具备普通网卡与高速 RoCE 网卡，高速口连入同一 RoCE VLAN。

本例采用 SR-IOV 方式为每个 Pod 分配一个高速网卡的虚拟功能（VF），由该 VF 在 Pod 内同时提供 Linux 接口 `net1` 与关联的 RDMA 端口。具体地址映射如下：

| 节点与 Pod | 默认网络接口 `eth0`（普通网卡） | RoCE 附加接口 `net1`（高速网卡 VF） |
| --- | --- | --- |
| `gpu-h0` / `default/rank-0` | `10.244.0.10` | `192.168.50.11/24` |
| `gpu-h1` / `default/rank-1` | `10.244.1.10` | `192.168.50.21/24` |

![gpu-h0 上的 rank-0 和 gpu-h1 上的 rank-1 各有 eth0 和 net1。上方蓝线经普通网口连接两端 eth0。下方青线经高速网卡的物理端口和同一 RoCE VLAN 连接两端 VF 的 RDMA 端口。灰虚线表示 net1 和本地 VF 的对应关系。](images/fig-13-01-01-default-and-roce-networks.png)

*图 13-01-01：同一对 Pod 的双网络通信全景。上方默认网络承载作业协调与 DNS 流量；下方 RoCE VLAN 承载与 `net1` 对应的 VF 硬件 RDMA 数据。图中只画出每节点本次使用的一张高速网卡，双向箭头表示可双向建连。*

## 给 Pod 接上 `net1` 后，流量一定走 RDMA 吗？

答案是否定的。给 Pod 装上第二张高速网口，只意味着多了一条可选出口；**数据包最终走哪张网卡、用什么协议，取决于“目的地址 + Pod 路由表 + 应用层协议选择”三者的组合**。

以发起端 Pod 保留默认路由（到对端默认 IP 走 `eth0`、到附加网段 `192.168.50.0/24` 走直连路由 `net1`）为例，三种常见条件会走出三条截然不同的路径：

| 目的地址与路由条件 | 应用层选用的传输方式 | 实际经过的物理与协议路径 |
| --- | --- | --- |
| 目标为对端默认 IP（`10.244.1.10`），路由出口 `eth0` | 普通 TCP Socket | 走普通网卡与集群默认网络的 TCP 连接 |
| 目标为对端附加 IP（`192.168.50.21`），路由出口 `net1` | 普通 TCP Socket | 虽走了高速网卡与 RoCE 以太网，但仍跑在内核 TCP 协议栈上 |
| 目标为对端附加 IP，两端显式打开与 `net1` 对应的 RDMA 设备与端口 | RDMA 建连与读写 | 真正跑在指定 VF 端口之间的硬件 RoCEv2 RDMA 路径 |

> [!warning] 常见误区：看到 `net1` 有流量 ≠ 正在跑硬件 RDMA
> 许多训练或推理框架在初始化失败时会自动回退到 Socket（TCP）模式。即使流量从高速网口 `net1` 发出，走内核 TCP 依然吃满 CPU 且无法享受显存直达。确认走上硬件 RoCE，必须核对进程日志选中的 RDMA 设备、端口及关联的 GID。

线上快速核对 Pod 的双套地址，只需对照两个控制面字段：`.status.podIP` 记录默认主网络 IP，而 Multus 写回的 `k8s.v1.cni.cncf.io/network-status` 注解记录包括 `net1` 在内的全部接口与 IP。再把这两组地址与进程日志里的目标 IP 和 RDMA 设备名一对照，流量去向就水落石出。

> [!summary] 核心结论
> Kubernetes 默认网络 `eth0` 为控制面与服务发现而生，附加高速网络 `net1` 为训练同步与推理 KV 迁移而生。**Multus** 把两者拼进同一个 Pod，但唯有“附加网 IP + 对应 RDMA 设备与端口”同时对准，应用才真正跑在硬件 RDMA 路径上。

那么，当一个 Pod 声明“我要接一张叫 `net1` 的 RoCE 网络”时，Multus 怎么知道该调用哪个插件、配哪段 IP、绑定哪张网卡设备？下一篇我们沿容器启动流程拆透 NAD 与 CNI 委派链。

> [!info]- 参考资料
> - **Kubernetes 网络与 Pod 模型**：[Kubernetes v1.36 网络模型](https://v1-36.docs.kubernetes.io/docs/concepts/services-networking/)、[Kubernetes v1.36 Pod 的网络共享](https://v1-36.docs.kubernetes.io/docs/concepts/workloads/pods/)、[CNI 规范](https://www.cni.dev/docs/spec/)
> - **Multus 多网络接入**：[Multus 多接口示例](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/how-to-use.md)
> - **RoCE 接入与 Linux 路由**：[NVIDIA Network Operator v26.7.1 的 SR-IOV RDMA 示例](https://docs.nvidia.com/networking/display/kubernetes2671/quick-start/sriov-network-rdma.html)、[NVIDIA DOCA 3.5.0 的 RoCE 与 GID 说明](https://networking-docs.nvidia.com/doca/archive/3-5-0/rdma-over-converged-ethernet)、[Linux 路由查询](https://man7.org/linux/man-pages/man8/ip-route.8.html)
