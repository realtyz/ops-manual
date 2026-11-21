# Operator 组件交付与 GPUDirect RDMA 验证

一对双节点 Pod 已经各自分到一张 H100 和一个高速 VF，也完成了 `net1` 地址与路由验证。接下来要跨过集群交付的最后一道硬门槛：**怎么证明两边的硬件通道真正跑通了 GPUDirect RDMA（GDR），而不是停留在“组件看起来都 Running”？**

工程上最可靠的做法，是先厘清 GPU Operator 与 **NVIDIA Network Operator** 各自交付的软件边界，再在同一对 VF 端口上跑一轮**主机内存 RDMA**与一轮**GPU 显存 RDMA**的对照实验。

## GPU 侧与 NIC 侧的双 Operator 协作分工

第 12 章已经拆解了 GPU Operator 如何用 `ClusterPolicy` 管好显卡软件栈。在高速网络侧，**NVIDIA Network Operator** 扮演对称角色。

它通过 `NicClusterPolicy`（全局策略）、`NicNodePolicy`（节点组策略）和子组件 SR-IOV Network Operator，自动化交付网卡驱动、Multus、CNI、NV-IPAM 与设备插件。

在 **Kubernetes `v1.36`、GPU Operator `v26.7.2`、Network Operator `v26.7.1`** 与 **containerd `2.3.0`** 基线下，两侧组件的交付分工与节点核对重点如下：

| 交付组件 | 策略入口与执行方 | 目标节点必须核对的结果 |
| --- | --- | --- |
| **GPU 驱动、Toolkit 与 GPU Device Plugin** | GPU Operator 的 `ClusterPolicy`（预装驱动池由节点团队维护） | 节点实际加载的 GPU 驱动、Toolkit 状态与容器拿到的 GPU |
| **NIC / RDMA 驱动** | 发行版兼容驱动，或由 Network Operator 在 `NicClusterPolicy` / `NicNodePolicy.spec.ofedDriver` 托管 DOCA-OFED | 主机实际加载的网卡与 RDMA 模块、版本、端口与 PCI BDF |
| **Multus、通用 CNI 与 NV-IPAM** | `NicClusterPolicy.spec.secondaryNetwork.multus`、`cniPlugins` 与 `spec.nvIpam` | 运行时生效的 CNI 配置、节点插件二进制、NV-IPAM 服务与地址池 |
| **VF 切分、库存发布与 NAD** | Helm 开启 `sriovNetworkOperator.enabled: true`，由 SR-IOV Network Operator 协调 | 节点策略生成的 VF、Node 扩展资源库存与自动渲染的 NAD |

部署时有两点容易踩坑：
- **避免重复托管同一 PF**：`NicNodePolicy` 也能按节点组部署 SR-IOV 或 RDMA Shared Device Plugin。给同一个 PF 选定唯一管理方后，务必检查 DaemonSet 覆盖，防止两个插件抢管同一批设备。
- **共用兼容的 NFD**：集群已有驱动、Multus 或 NFD 时，须先认清现网提供方。双 Operator 通常共用一套兼容的 NFD，并核对 `NodeFeatureRule` API 与双方依赖的 PCI 厂商标签。

查看控制面协调进度时，可读取 `NicClusterPolicy.status.conditions[]`（如 `type: MultusCNIReady`、`status: "True"`、`reason: ComponentReady`），并用 `observedGeneration` 对照 `metadata.generation` 确认它对应最新配置。

与第 12 章同理，集群级汇总 `Ready` 只覆盖策略内受管组件。节点实态与 Pod 接入结果仍须下沉核对。

## 第一轮验证：在指定 VF 上建立主机内存 RDMA 基线

在动用 GPU 显存之前，先把网络本身验证干净。在双端 Pod 中，先用 `rdma link show` 确认哪个 RDMA 设备和端口对应本机 `net1`，再用 `ibv_devinfo` 核对端口状态为 `PORT_ACTIVE` 且链路类型为 `Ethernet`，并确认到对端附加 IP `192.168.50.11` 的路由指向 `net1`。

随后，两端使用同一版本的 `perftest 26.07.8`（镜像内具备工具、用户态驱动与设备/锁页权限），执行主机内存 `ib_write_bw` 基准测试：

```sh
# 1) 在服务端 Pod（gpu-h0 上的 rank-0）启动接收端
#    -d/-i 填入本端对应 net1 的 RDMA 设备与端口号；-R 使用 RDMA CM 建连
ib_write_bw -d "$RDMA_DEV" -i "$RDMA_PORT" -R -s 65536 -n 1000 --report_gbits

# 2) 在客户端 Pod（gpu-h1 上的 rank-1）发起单向 Write（载荷方向：rank-1 → rank-0）
ib_write_bw -d "$RDMA_DEV" -i "$RDMA_PORT" -R -s 65536 -n 1000 --report_gbits 192.168.50.11
```

这组命令以 $64\text{ KiB}$（`65536 B`）消息长度连续发送 1000 次单向 RDMA Write，并以 `Gb/s` 汇报带宽。只要两端正常退出且速率落在物理链路的合理区间，就证明**这条 VF 通道、RoCE 交换网、GID 与主机内存 RDMA 注册已经全通**。

> [!tip] 工程直觉：读硬件计数器要分清 PF 与 VF
> 若用网卡端口计数器佐证测试流量，务必注明读取的是 PF 总计数还是 VF 独立计数，并记录采样窗口。PF 总计数器会混入同一物理口下其他 VF 的并发流量。

## 第二轮验证：只替换两端缓冲区为 GPU 显存

第一轮跑通后，固定**同一对 VF 端口、同一传输方向、同一消息大小与同一工具构建**，只把两端收发缓冲区从主机内存换成 GPU 显存。这就是控制变量法的精髓——如果这一轮失败，根因立刻精准锁定在“显存导出与注册链路”上。

当前 Kubernetes GPU 集群支持两条显存注册路径，前置条件差异明显：

| 显存注册路径 | 工作机制 | 核心前置条件与 Operator 配置要点 |
| --- | --- | --- |
| **DMA-BUF（推荐现代路径）** | CUDA 导出显存的 DMA-BUF 文件描述符，RDMA 栈直接导入注册 | Linux 内核 `≥ 5.12`、开源 GPU 内核模块、容器内 CUDA `≥ 11.7`，配兼容的 GPU/NIC 驱动与应用；满足驱动条件时可直接用发行版 NIC 驱动 |
| **`nvidia-peermem`（传统路径）** | 由内核模块 `nvidia_peermem` 协助 RDMA 栈注册 GPU 虚拟地址 | 需配合 DOCA-OFED 或 MLNX_OFED；Operator 托管驱动时启用 `driver.rdma.enabled=true`，预装宿主 OFED 时核对 `driver.rdma.useHostMofed` |

使用支持 CUDA 的 `perftest` 构建复测，只需在两端命令中追加显存参数（其中 `0` 为容器内可见的 CUDA 设备号，需提前与分配结果核对）：

```sh
# 方式 A：验证 DMA-BUF 路径（两端均追加 --use_cuda=0 --use_cuda_dmabuf）
ib_write_bw -d "$RDMA_DEV" -i "$RDMA_PORT" -R -s 65536 -n 1000 \
  --use_cuda=0 --use_cuda_dmabuf --report_gbits 192.168.50.11

# 方式 B：验证 nvidia-peermem 传统路径（去掉 --use_cuda_dmabuf）
ib_write_bw -d "$RDMA_DEV" -i "$RDMA_PORT" -R -s 65536 -n 1000 \
  --use_cuda=0 --report_gbits 192.168.50.11
```

如果主机内存轮顺利通过，显存轮却在某一端报注册失败，排查焦点非常集中：在报错端先区分是 **CUDA 显存分配失败**、**DMA-BUF 导出失败**，还是 **RDMA 导入注册失败**，再逐一核对容器 CUDA 设备号、工具构建选项、宿主机内核/驱动模块以及第 06 篇检查的 PCIe 路径。

两端显存轮跑通，就拿到了这套设备对与注册路径下 GPUDirect RDMA 可用的直接证据（无论是训练梯度同步还是 PD 分离推理的跨机 KV Cache 直传，都建立在这条通道之上）。

> [!note] 跨 Operator 协作与支持矩阵核对
> 同时部署 GPU Operator `v26.7.2` 与 Network Operator `v26.7.1` 时，生产评审务必做两项交叉核对：
> 1. **共用 NFD 对齐**：检查两份 chart 各自声明的 NFD 依赖版本与实际渲染镜像，确认运行中的 NFD 同时满足双方所需的 `NodeFeatureRule` API 与 PCI 厂商标签。
> 2. **GDR 组合支持矩阵**：按所选注册方式（DMA-BUF 或 DOCA-OFED + `nvidia_peermem`），同时核对 GPU 型号、网卡与固件、Linux 内核及双 Operator 平台支持矩阵，并在目标节点池完成双轮实测验收。

> [!summary] 核心结论
> 验收 Pod 高速网络切忌一步跳到大型训练框架里盲猜。先核对双 Operator 组件状态，再用 `perftest` 跑通**主机内存 RDMA**立住网络基线，最后只换缓冲区跑**显存 GDR**——哪一轮掉链子，根因就在哪一层。

这套“逐层控制变量”的思路不仅适用于上线验收，更是生产故障定位的利器。下一篇我们结合真实的 `kubectl describe`、`network-status`、`ip` 与 `rdma` 输出沉淀五层排障清单，并彻底厘清一个高频安全盲区：**为什么 Kubernetes 默认的 `NetworkPolicy` 管不到 `net1`？**

> [!info]- 参考资料
> - **GPU Operator 与 Network Operator 平台矩阵及 GDR 文档**：[NVIDIA GPU Operator v26.7.2 的 GPUDirect RDMA 说明](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-rdma.html)、[NVIDIA GPU Operator v26.7.2 的 GDR 前置条件](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-rdma.html#gpudirect-rdma-prerequisites)、[NVIDIA 的 GPU 缓冲区验证与输出](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-rdma.html#verifying-the-installation-by-performing-a-data-transfer)、[GPU Operator v26.7.2 平台支持](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/platform-support.html)、[GPU Operator v26.7.2 的 GDR 平台列表](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/platform-support.html#support-for-gpudirect-rdma)、[GPU Operator v26.7.2 的 chart 依赖](https://github.com/NVIDIA/gpu-operator/blob/v26.7.2/deployments/gpu-operator/Chart.yaml)
> - **Network Operator v26.7.1 配置、状态与支持矩阵**：[Network Operator v26.7.1 平台与组件矩阵](https://docs.nvidia.com/networking/display/kubernetes2671/platform-support.html)、[Network Operator v26.7.1 的 NFD 前置条件](https://docs.nvidia.com/networking/display/kubernetes2671/platform-support.html#prerequisites)、[Network Operator v26.7.1 的 GDR 平台列表](https://docs.nvidia.com/networking/display/kubernetes2671/platform-support.html#support-for-gpudirect-rdma)、[Network Operator 的 Helm 配置和 NFD 标签](https://docs.nvidia.com/networking/display/kubernetes2671/customizations/helm.html)、[NicClusterPolicy 与 NicNodePolicy 的组件范围](https://docs.nvidia.com/networking/display/kubernetes2671/customizations/nic-node-policy.html)、[Network Operator v26.7.1 的状态字段](https://docs.nvidia.com/networking/display/kubernetes2671/life-cycle-management.html)、[NVIDIA v26.7.1 的 SR-IOV RDMA 双端验证](https://docs.nvidia.com/networking/display/kubernetes2671/quick-start/sriov-network-rdma.html)
> - **CUDA GDR 与 perftest 参数参考**：[CUDA GPUDirect RDMA PCIe 约束](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems)、[perftest 26.07.8 选项](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)、[perftest 26.07.8 的 RDMA CM 与报告参数](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_parameters.c)
