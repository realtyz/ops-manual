
两个 Pod 已经通过附加网络获得 VF 和地址，也各自拿到一张 GPU。第一次试验让网卡传输主机内存中的数据，第二次试验把两端的缓冲区换成 GPU 显存。**GPUDirect RDMA（GDR）** 提供后一条路径所需的 GPU 和网卡间直接访问能力。用这两轮传输作为交付检验，可以把组件协调、网络传输和显存注册的证据逐项对应起来。 [NVIDIA GPU Operator v26.7 的 GPUDirect RDMA 说明](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-rdma.html)

## GPU 与 NIC 两侧的组件交付

第 12 章用 `ClusterPolicy` 说明 GPU Operator 怎样从配置协调 GPU 驱动、Container Toolkit 和设备插件。**NVIDIA Network Operator** 在 NIC 侧做相应的组件交付，例如部署附加网络插件、IPAM、设备插件和可选的网卡驱动。主例以 Kubernetes v1.36、GPU Operator v26.7.2 和 Network Operator v26.7.1 阅读接口，沿用 Ubuntu 24.04 LTS x86_64 和 containerd 2.3。配置入口及其在节点上的输出如下。 [Network Operator v26.7.1 平台与组件矩阵](https://docs.nvidia.com/networking/display/kubernetes2671/platform-support.html) · [GPU Operator v26.7 平台支持](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/platform-support.html)

| 要交付的组件 | 设置和执行方 | 目标节点要核对的结果 |
| --- | --- | --- |
| GPU 驱动、Toolkit、GPU Device Plugin | GPU Operator 的 `ClusterPolicy` 管理启用项。采用预装驱动时由节点团队维护驱动。 | 实际加载的 GPU 驱动、Toolkit 和插件在目标节点的状态、工作容器获得的 GPU。 |
| NIC/RDMA 驱动 | 可用兼容的发行版驱动。需要由 Network Operator 管理 DOCA-OFED 时，可在 `NicClusterPolicy` 配置驱动，或用 `NicNodePolicy.spec.ofedDriver` 与节点选择器按节点组交付。 | 主机实际加载的网卡和 RDMA 模块、版本、端口与 PCI 地址。 |
| Multus、通用 CNI 插件、NV-IPAM | `NicClusterPolicy.spec.secondaryNetwork.multus`、`secondaryNetwork.cniPlugins`、`spec.nvIpam` 管理启用项。 | 运行时选中的 CNI 配置、实际可执行文件、NV-IPAM 节点服务和地址池。 |
| VF 配置、资源发布和 NAD | 主例用 Helm 的 `sriovNetworkOperator.enabled: true` 启用 SR-IOV Network Operator。该子 Operator 协调 VF 配置、设备插件与网络对象。 | 节点策略对应的 VF、Node 上的扩展资源、实际生成的 NAD。 |

`NicNodePolicy` 也可按节点组管理 RDMA Shared Device Plugin 和 SR-IOV Device Plugin。为同一个 PF 选定管理方式后，检查对应 DaemonSet 的节点覆盖和实际资源选择，避免另一个插件实例重复管理同一批设备。已有驱动、Multus 或 NFD（Node Feature Discovery，节点特征发现）时，先辨认现网的提供方。两个 Operator 共用一套兼容的 NFD，并核对 `NodeFeatureRule` API 和双方用到的 PCI 厂商标签。 [Network Operator 的 Helm 配置和 NFD 标签](https://docs.nvidia.com/networking/display/kubernetes2671/customizations/helm.html) · [NicClusterPolicy 与 NicNodePolicy 的组件范围](https://docs.nvidia.com/networking/display/kubernetes2671/customizations/nic-node-policy.html)

例如 `NicClusterPolicy.status.conditions[]` 中，`type: MultusCNIReady` 表示对应组件条件。`status: "True"`、`reason: ComponentReady` 表示该受管组件达到协调就绪条件，可用 `observedGeneration` 和对象的 `metadata.generation` 核对这条状态针对哪次配置。汇总状态 `Ready` 只涵盖配置中的受管组件。验收 `gpu-h0` 上的 Pod 时，继续核对目标节点组件、已加载驱动和该 Pod 的接入结果。 [Network Operator v26.7.1 的状态字段](https://docs.nvidia.com/networking/display/kubernetes2671/life-cycle-management.html)

## 在指定 VF 上建立主机内存 RDMA 基线

用主例中的 `rank-0` 和 `rank-1`，在两端通过 `rdma link show` 确认哪一个 RDMA 设备和端口对应本地 `net1`，用 `ibv_devinfo` 核对端口状态与 Ethernet 链路类型。然后从客户端检查到服务端附加地址 `192.168.50.11` 的路由。工具使用相同的 perftest 构建。命令里的 `RDMA_DEV` 和 `RDMA_PORT` 需要在各端按本地探测结果赋值。假定镜像已提供工具、用户态驱动和相应的设备及锁页权限。 [NVIDIA v26.7.1 的 SR-IOV RDMA 双端验证](https://docs.nvidia.com/networking/display/kubernetes2671/quick-start/sriov-network-rdma.html) · [perftest 26.07.8 选项](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)

```sh
# rank-0（gpu-h0）先启动服务端
ib_write_bw -d "$RDMA_DEV" -i "$RDMA_PORT" -R -s 65536 -n 1000 --report_gbits

# rank-1（gpu-h1）再启动客户端，载荷方向为 rank-1 → rank-0
ib_write_bw -d "$RDMA_DEV" -i "$RDMA_PORT" -R -s 65536 -n 1000 --report_gbits 192.168.50.11
```

命令是教学示意。`-R` 用 RDMA CM（RDMA Connection Manager，连接管理器）建立测试用队列对，客户端位置参数给出服务端附加 IP。设置 `65536 B`（`64 KiB`）消息和 `1000` 次测试迭代，保持默认单方向。记录两端实际设备和端口、缓冲区类型、退出状态以及报告的 Gb/s 口径。一次成功支持这组参数下的主机内存 RDMA Write 路径可用。若用端口计数佐证，要注明计数归属是 PF 还是 VF、采样区间及是否有并发流量。PF 总计数可能混入其他 VF 的活动。 [perftest 26.07.8 的 RDMA CM 与报告参数](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_parameters.c)

## 将两端缓冲区换成 GPU 显存

普通 RDMA 先向网卡注册主机内存。要让 NIC 直接访问显存，应用先创建 CUDA 显存缓冲区，再将它注册给指定 RDMA 设备。主机侧需要兼容的设备、驱动和可用的 PCIe 路径。工作容器需要能使用分到的 GPU 与 RDMA 设备，能够加载 CUDA 用户态、Verbs 库及网卡对应的用户态驱动。测试程序也需要支持选择的注册方式。 [NVIDIA GPU Operator v26.7 的 GDR 前置条件](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-rdma.html#gpudirect-rdma-prerequisites) · [CUDA 13.1.2 的 PCIe 约束](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems)

| 显存注册方式 | 过程与适用条件 |
| --- | --- |
| DMA-BUF | 应用用 CUDA 导出显存缓冲区的 DMA-BUF 文件描述符，再由支持该接口的 RDMA 栈导入并注册。GPU Operator v26.7 的文档要求 Linux 内核 ≥ 5.12、开放 GPU 内核模块和工作负载内 CUDA ≥ 11.7，另外要有支持的 GPU、NIC 驱动和应用。驱动能力满足时可用发行版 NIC 驱动与 RDMA 用户态软件包。 |
| `nvidia-peermem` | 应用通过支持的 RDMA 栈注册 GPU 虚拟地址，加载的 `nvidia_peermem` 模块提供 GPU 对等内存支持。GPU Operator v26.7 的这条路径要求 MLNX_OFED 或 DOCA-OFED。GPU 驱动由 Operator 管理时可用 `driver.rdma.enabled=true` 启用该模块，预装 MLNX_OFED 的情况按文档核对 `driver.rdma.useHostMofed`。 |

在具备对应条件的同一设备对上，用带 CUDA 支持的 perftest 构建复测。DMA-BUF 轮在两端分别加 `--use_cuda=0 --use_cuda_dmabuf`。其中 `0` 是各自工作容器内的 CUDA 设备编号，需先与实际分配结果核对。legacy 轮用 `--use_cuda=0`，省去 `--use_cuda_dmabuf`。先用工具的帮助确认参数可用，固定两端的设备与端口、消息大小、迭代数、方向和工具版本。GPU Operator v26.7 给出的 GPU 验证原例采用 Macvlan + RDMA Shared Device Plugin。这里沿用其对缓冲区的验证方法，用于设计同一 VF 设备对的对照实验。 [NVIDIA 的 GPU 缓冲区验证与输出](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-rdma.html#verifying-the-installation-by-performing-a-data-transfer) · [perftest 26.07.8 的 CUDA 及 DMA-BUF 用法](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)

若主机内存轮成功，显存轮在一端报告注册失败，先在出错端区分 CUDA 分配、导出和 RDMA 导入，再查设备编号、测试构建、实际驱动和 PCIe 路径。两端显存轮成功时，证据覆盖所测的设备对、注册方式和方向。训练应用是否使用同一路径，还要看它实际选择的传输与设备。以上 VF 命令与条件用于教材推演，暂无实机传输结果。

## 发行组件与整套平台的支持条件

Network Operator v26.7.1 的平台矩阵建议两侧 Operator 共用 GPU Operator 的 NFD，同时列出 NFD v0.15.6–v0.17.0。GPU Operator v26.7.2 源码标签声明的 NFD 子 chart 依赖为 0.19.0。共用时应核对发行 chart 渲染出的实际镜像、所需 API 和运行中的节点标签，并确认组合支持。 [Network Operator v26.7.1 的 NFD 前置条件](https://docs.nvidia.com/networking/display/kubernetes2671/platform-support.html#prerequisites) · [GPU Operator v26.7.2 的 chart 依赖](https://github.com/NVIDIA/gpu-operator/blob/v26.7.2/deployments/gpu-operator/Chart.yaml)

GDR 也有组合口径。GPU Operator v26.7 平台列表的 Ubuntu 24.04 项列有 Network Operator 26.1.0。Network Operator v26.7.1 的 GDR 列表采用 GPU Operator v25.3.x 或更新、DOCA-OFED 和 `nvidia_peermem` 的条件。DMA-BUF 的独立机制条件来自 GPU Operator 的 GDR 专章。评审具体环境要按注册方式、GPU、NIC 和固件、内核、两侧驱动与 Operator 版本核对。主例版本说明对应对象与机制，所示 VF 组合的端到端支持仍需厂商确认和实机验证。 [GPU Operator v26.7 的 GDR 平台列表](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/platform-support.html#support-for-gpudirect-rdma) · [Network Operator v26.7.1 的 GDR 平台列表](https://docs.nvidia.com/networking/display/kubernetes2671/platform-support.html#support-for-gpudirect-rdma)
