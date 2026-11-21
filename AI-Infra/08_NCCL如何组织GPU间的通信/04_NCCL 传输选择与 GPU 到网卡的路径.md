# NCCL 传输选择与 GPU 到网卡的路径

上一篇的逻辑环里，每条相邻有向边承载着一模一样的算法载荷。可一旦落到真实的两台服务器（`gpu-a` 与 `gpu-b`），情况立刻分化：同机相邻卡只需在主板内部走总线，跨机对端卡却必须穿过网卡与交换网络。

更让人意外的是：**哪怕服务器硬件、GPU 数量和每张卡的放置一动不动，仅仅改变逻辑环的邻居排列顺序，跨机网卡的单向流量就可能瞬间翻倍**。本篇沿着 NCCL 的物理拓扑探测，把逻辑边逐条映射到真实硬件通路上。

## 从拓扑探测到五类传输实现

NCCL 2.27.3 初始化时会扫描本机全部 GPU、PCIe 交换芯片、CPU NUMA 与高速网卡，计算设备间的物理路径代价，进而搜索 Ring/Tree 通信图并在全组汇总，为每条逻辑通道安排具体的收发邻居。

针对每一对逻辑邻居，NCCL 会**分别在发送侧与接收侧**检验可用的传输实现，按优先级选出数据搬运路径：

| 通信范围与选中传输 | 载荷在物理硬件上的实际移动路径 |
| --- | --- |
| **同机 · P2P** | 发送卡经 NVLink 或 PCIe Switch 直接读写机内邻居卡的显存，无需经过主机内存 |
| **同机 · SHM** | 两张卡经主机内存中的共享缓冲区中转交换数据（P2P 不可用或受限时的回退） |
| **跨机 · NET `IB` + GDR** | 本地高速网卡经 GPUDirect RDMA 直接对已注册的 GPU 显存执行 DMA 读写 |
| **跨机 · NET `IB` + 主机缓冲** | GPU 与主机内存之间先做显存拷贝，再由高速网卡对已注册的主机缓冲执行 RDMA |
| **跨机 · NET `Socket`** | 数据经主机缓冲与内核 TCP/IP 协议栈，从指定普通 IP 网口收发 |

注意区分两个同名概念：第 01 篇介绍的点对点 API（`ncclSend` / `ncclRecv`）属于**接口语义**，规定哪两个成员直接配对；上表里的 **P2P** 则是**底层传输实现**，集合操作（如 AllReduce）中的机内边同样会优先走 P2P 直达显存。

另外，NCCL 内置网络后端名称叫 `IB`，但只要网卡支持 Verbs 接口，它既能跑原生 InfiniBand，也能跑封装在 UDP/IP 里的 **RoCEv2**。至于跨机能否启用 **GDR（GPUDirect RDMA）**，则依赖 GPU 与网卡的 PCIe 拓扑亲和、驱动及显存映射机制（如 Linux DMA-BUF 或 `nvidia-peermem`）共同就绪。

## 同一条 Ring 的机内边、跨机边与单侧回退

在基准环 `0 → 1 → 2 → 3 → 0` 中，`Rank 0/1` 落在 `gpu-a`，`Rank 2/3` 落在 `gpu-b`。观察图 08-04-01 中相邻的两条有向边，可以直观看到三种流量如何各行其道：

- **机内边 `0 → 1`（绿色）**：两张 GPU 挂在 `gpu-a` 的同一颗 PCIe Switch 下，选中 PCIe P2P 后，载荷直接经交换芯片从源显存进入目标显存。
- **跨机边 `1 → 2`（橙色）**：假设收发两端均开启 GDR，载荷从 `gpu-a` 的 GPU 1 显存经本地 PCIe Switch 进入 `mlx5_0:1` 网卡，穿过 RoCEv2 交换网络抵达 `gpu-b` 的对端网卡，再经对端 PCIe Switch 直接写进 GPU 0 显存。
- **引导流量（蓝色虚线）**：两端 `eno1` 接口通过独立的 TCP 网络交换 Bootstrap 元数据，与高速数据面完全隔离。

![gpu-a 的 Rank 0 经本机 PCIe Switch 向 Rank 1 传数据。Rank 1 的载荷经两端 PCIe Switch、RoCE 网卡和中间的 RoCEv2 网络到达 gpu-b 的 Rank 2。两端 eno1 经独立的 TCP 网络交换引导信息。](images/fig-08-04-01-ring-data-paths.png)

*图 08-04-01：双机四卡基准环中相邻两条边的物理路径对照（绿色为机内 PCIe P2P 载荷，橙色为跨机 RoCEv2 GDR 载荷，蓝色虚线为 TCP 引导通道，其余环边已省略）。*

> [!warning] 常见误区：跨机边两端不必同时开启 GDR
> 一条跨机连接的发送侧和接收侧各自独立决定是否使用显存直达。假如发送节点因 PCIe 拓扑或 `NCCL_NET_GDR_READ` 策略回退到主机中转，而接收节点仍保留 GDR：发送端就会变成“GPU 1 显存 → 本地主机内存 → 本地网卡”，接收端则依然“对端网卡 → GPU 0 显存”。排障时必须把发送端和接收端的通道日志拆开逐一核对。

## 环顺序如何让单端口负载直接翻倍

现在把每张卡的 AllReduce 输入放大到训练常见规模 `64 MiB`，继续沿用四卡单通道等分块 Ring 模型。由上一篇公式，**每条有向环边**在一次操作中发送的算法载荷为：

$$V_{\text{edge}} = \frac{2(P - 1)}{P} \times S = \frac{2 \times 3}{4} \times 64\text{ MiB} = 96\text{ MiB}$$

两台服务器各只有一块 `200 Gb/s` 全双工 RoCEv2 网卡，单方向理论线速为 $B_{\text{link}} = 200 \div 8 = 25\text{ GB/s}$。全双工允许去程和回程并行，但**同一方向上若有多条逻辑环边共用同一物理网口，它们的发送字节必须累加**：

- **顺排环（`0 → 1 → 2 → 3 → 0`）**：`gpu-a → gpu-b` 方向仅有 `1 → 2` 一条跨机边，单向端口载荷为 `96 MiB`：

$$T_{\min,\text{seq}} = \frac{96 \times 2^{20}\text{ B}}{25 \times 10^9\text{ B/s}} = \frac{100{,}663{,}296\text{ B}}{25 \times 10^9\text{ B/s}} \approx 4.03\text{ ms}$$

- **交错环（`0 → 2 → 1 → 3 → 0`）**：每跳都在两台机器间来回折返，`gpu-a → gpu-b` 方向同时堆了 `0 → 2` 与 `1 → 3` 两条边，单向端口载荷翻倍至 `192 MiB`：

$$T_{\min,\text{interleaved}} = \frac{192 \times 2^{20}\text{ B}}{25 \times 10^9\text{ B/s}} = \frac{201{,}326{,}592\text{ B}}{25 \times 10^9\text{ B/s}} \approx 8.05\text{ ms}$$

| 逻辑通道环顺序 | `gpu-a → gpu-b` 跨机边 | `gpu-b → gpu-a` 跨机边 | 单方向网口载荷 | 名义线速耗时下界 |
| --- | --- | --- | ---: | ---: |
| `0 → 1 → 2 → 3 → 0` | `1 → 2` | `3 → 0` | `96 MiB` | `4.03 ms` |
| `0 → 2 → 1 → 3 → 0` | `0 → 2`、`1 → 3` | `2 → 1`、`3 → 0` | `192 MiB` | `8.05 ms` |

不难看出，为什么 NCCL 在拓扑搜索阶段会拼命把同机相邻卡排在逻辑环里紧挨着走完，再只留最少的边跨出机箱——每多出一条无谓的跨机折返边，跨机网口的理论时间下界就会成倍抬高。在多通道并行场景下，核算端口压力也按同样方法：先拆出每条通道分担的数据量，再按物理网口和方向分别累加。

## 对齐设备、网卡端口与 RoCEv2 GID

要确保 NCCL 准确选中预期的引导接口与高速网卡，动手配环境变量前先在两台主机上核清四层映射关系（因不同机器上网卡名常相同，记录时必须带上主机名）：

| 平面用途 | 逻辑端口与主机接口 | `gpu-a` 地址 | `gpu-b` 地址 |
| --- | --- | --- | --- |
| 默认 TCP 引导网 | `eno1` | `192.0.2.11` | `192.0.2.12` |
| RoCEv2 高速计算网 | `mlx5_0:1`（对应 `enp129s0f0`） | `198.51.100.11` | `198.51.100.12` |

在各节点依次执行下面几条只读检查，即可把 GPU 拓扑、RDMA 端口、活动 MTU 与本地 RoCEv2 GID 全部钉死：

```bash
# 1. 核对本机 GPU 之间及 GPU 与网卡的 PCIe 亲和矩阵
nvidia-smi topo -m

# 2. 将 RDMA 端口名对回 Linux 网卡接口，并检查链路层与活动 MTU
rdma link show mlx5_0/1
ibv_devinfo -d mlx5_0 -i 1

# 3. 核对 RoCEv2 GID 表项（逐一确认 GID 值、类型 RoCE v2 与绑定接口 enp129s0f0）
grep -v '^0000:0000:0000:0000:0000:0000:0000:0000$' /sys/class/infiniband/mlx5_0/ports/1/gids/*
cat /sys/class/infiniband/mlx5_0/ports/1/gid_attrs/types/3   # 期望输出: RoCE v2
cat /sys/class/infiniband/mlx5_0/ports/1/gid_attrs/ndevs/3   # 期望输出: enp129s0f0
```

在 `gpu-a` 上，上述检查应把 GID 值 `::ffff:198.51.100.11`、类型 `RoCE v2` 与接口 `enp129s0f0` 对应到同一个本地索引；`gpu-b` 则按自身地址核对自己的表（两台机器的本地 GID 索引号不必相同）。

从 NCCL 2.21 起，推荐放弃硬编码脆弱的 `NCCL_IB_GID_INDEX`，转而用地址族、网段与 RoCE 版本让 NCCL 动态筛出正确 GID。NCCL 2.27.3 的典型配置如下：

```bash
export NCCL_SOCKET_IFNAME='=eno1'        # 前缀 = 表示精确匹配，锁定引导与 Socket 接口
export NCCL_IB_HCA='=mlx5_0:1'           # 精确锁定高速计算网所使用的 RDMA 端口
export NCCL_IB_ADDR_FAMILY=AF_INET       # 仅匹配 IPv4 映射的 RoCE GID
export NCCL_IB_ADDR_RANGE=198.51.100.0/24 # 将动态 GID 候选限定在计算网段内
export NCCL_IB_ROCE_VERSION_NUM=2        # 强制选中 RoCEv2，避开同 IP 的 RoCEv1 孪生表项
```

> [!tip] 工程直觉：别拿 NCCL 变量去管 PD 分离的 KV 迁移网卡
> 上述 `NCCL_IB_*` 变量只约束 NCCL 自身的集合与点对点通道。当在线推理采用 PD 分离并由 **NIXL + UCX** 执行跨节点 KV Cache 迁移时，底层选卡往往由 UCX 侧环境变量（如 `UCX_NET_DEVICES`、`UCX_TLS`）接管。混部或排障时务必按通信库分别核对选口配置。

> [!note] 版本与平台说明：SHM 分配、跨机 NVLink 与多 NIC 选项
> - **同机 SHM 机制**：NCCL 2.24 起在驱动 CUDA $\ge$ 12.6、Runtime $\ge$ 12.2 且支持 `cuMem` 主机分配时优先采用该机制而非传统 `/dev/shm`；自 2.26.5 起具备运行时探测并在不支持时自动回退 `/dev/shm`。
> - **多 NIC 与跨节点 NVLink 域**：在每节点配备多张高速网卡（多 Rail）的集群上，`NCCL_CROSS_NIC` 决定同一条 Ring 或 Tree 是否允许在不同节点跨用不同编号的网卡；在机柜级 Scale-up NVLink 域或启用外部网络插件的平台上，NCCL 的候选跨机路径会优先切向多节点 NVLink 或对应插件。

配好环境变量只是“告诉 NCCL 我们的期望”，它运行时到底有没有按预期选中 `mlx5_0:1`、两端是不是都走了 `/GDRDMA`、卡住时又卡在哪个节点？下一篇我们直接拆解 `NCCL_DEBUG` 日志。

> [!info]- 参考资料
> - **NCCL 拓扑初始化与传输选择源码**：[`src/init.cc`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/init.cc)、[`src/transport.cc#L14`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport.cc#L14)、[`src/transport/net.cc`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net.cc)、[`src/graph/paths.cc#L388`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/graph/paths.cc#L388)
> - **NCCL 网络、GDR、GID 与 SHM 配置指南**：[`NCCL_NET`](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-net)、[`NCCL_SOCKET_IFNAME` / `NCCL_IB_HCA`](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-socket-ifname)、[`NCCL_IB_GID_INDEX` 与动态 GID](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-ib-gid-index)、[`NCCL_CROSS_NIC`](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-cross-nic)、[GPUDirect 排障](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html#gpu-direct)、[RoCE 排障](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html#rdma-over-converged-ethernet-roce)、[`cuMem` 主机共享内存排障](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html#cumem-host-allocations)
> - **硬件拓扑、端口与 RoCE GID 手册**：[NVIDIA SMI Manual](https://docs.nvidia.com/deploy/nvidia-smi/index.html)、[iproute2 v6.12.0 `rdma-link(8)`](https://github.com/iproute2/iproute2/blob/v6.12.0/man/man8/rdma-link.8)、[rdma-core v57.0 `ibv_devinfo(1)`](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/man/ibv_devinfo.1)、[NVIDIA MLNX_OFED RoCE & GID Table](https://docs.nvidia.com/networking/display/nvidia-mlnx-ofed-documentation-v24-07-0-6-1-0.0.pdf)
> - **AllReduce 环边载荷模型**：[nccl-tests v2.17.9 Performance Formulas](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/doc/PERFORMANCE.md#allreduce)
