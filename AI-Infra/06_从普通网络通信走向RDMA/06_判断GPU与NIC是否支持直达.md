# 如何判断一对 GPU 与 NIC 支持直达

`nvidia-smi topo -m` 显示 GPU 和 NIC 是 `PIX`，显存注册还是可能失败。拓扑矩阵描述了设备之间经过的连接，接下来还要判断桥如何转发 PCIe 事务、设备能否用所需地址访问彼此，以及 GPU 与 RDMA 驱动能否建立映射。下面以裸机 Linux 上的传统 PCIe 路径为例。 [NVIDIA SMI · Topology](https://docs.nvidia.com/deploy/nvidia-smi/) · [CUDA 13.1.2 · Supported Systems](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems)

## 先用 BDF 把拓扑矩阵还原成设备树

`nvidia-smi topo -m` 里的 `PIX` 表示经过一个 PCIe Switch，`PHB` 表示经过 PCIe Host Bridge，`SYS` 表示还经过 NUMA 节点之间的处理器互联。先用 `nvidia-smi --query-gpu=index,uuid,pci.bus_id --format=csv` 将 GPU 序号对回 PCI 地址，再从 `/sys/class/infiniband/<设备>/device` 获取 RDMA 设备对应的 PCI 路径。这里的 PCI 地址即 BDF，由域、总线、设备和功能组成。拿两端点在 `lspci -t` 中向上找共同祖先。NUMA 节点编号相同也要核对两设备的上游 Root Complex 和桥。 [NVIDIA SMI · GPU 查询与拓扑](https://docs.nvidia.com/deploy/nvidia-smi/) · [CUDA 13.1.2 · Supported Systems](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems)

| 路径 | 载荷可能经过的部件 | 如何判断 |
| --- | --- | --- |
| 同一 PCIe Switch | GPU 和 NIC 各连到 Switch 的下游端口，事务可在 Switch 内转发。 | 对照两边的 BDF 和桥的 ACS 控制位。 |
| 经过 PCIe Host Bridge | 事务上行至主桥，再到目标所在 PCIe 分支。 | 查平台是否支持所需转发。CUDA 指南给出的传统路径以共享上游 Root Complex 为重要前提，并提到一些处理器上对等读带宽受限。 |
| 跨 Socket | 事务还要经过 Socket 间互联。 | 按平台支持核对可达性。即使能传，也要分别验证 NIC 读 GPU 和 NIC 写 GPU 的表现。 |

这些类别给出排查方向，具体能否转发取决于主桥、交换层和设备配置。 [CUDA 13.1.2 · Supported Systems](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems) · [Linux 内核 · PCI Peer-to-Peer DMA](https://docs.kernel.org/driver-api/pci/p2pdma.html)

## ACS 和 IOMMU 分别影响路径和地址

**ACS**（PCIe Access Control Services）是 PCIe 层的访问控制能力。对等请求和完成的重定向位会影响事务是否被送上游。以某个 Switch 下游桥为例，用 `lspci -vv -s 0000:81:02.0` 读取 `ACSCtl` 中的 `ReqRedir` 和 `CmpltRedir`。`ACSCap` 列出设备具备哪些控制能力，`ACSCtl` 才显示控制位的状态。`SrcValid+` 只说明 Source Validation 的状态，要判断重定向仍需读对应的控制位和路径上其余桥。 [pciutils v3.13.0 · ACS 字段](https://github.com/pciutils/pciutils/blob/v3.13.0/ls-ecaps.c) · [Linux 内核 · PCI Peer-to-Peer DMA](https://docs.kernel.org/driver-api/pci/p2pdma.html)

以下是两次**独立的教学读数**。假定同一台裸机上的 GPU 0 是 `0000:82:00.0`，`mlx5_0` 是 `0000:83:00.0`，两者接在同一 Switch 的下游桥 `0000:81:01.0` 与 `0000:81:02.0` 后面。

| 对同一对设备的读数 | `topo -m` 的距离 | NIC 侧下游桥的 `ACSCtl` 摘录 | 下一步判断 |
| --- | --- | --- | --- |
| 条件甲 | `PIX` | `SrcValid+ ReqRedir- CmpltRedir-` | 该桥显示这两个重定向位关闭，继续核对其余桥和 DMA 地址条件。 |
| 条件乙 | `PIX` | `SrcValid+ ReqRedir+ CmpltRedir-` | 请求可能被送上游。按该方向的事务发起方核对完整路径，以及上游是否能转回目标。 |

**IOMMU** 管设备 DMA 的地址转换与隔离。CUDA 13.1.2 对其所述的传统 GPUDirect RDMA 路径要求设备看到一致的地址，因而要核对实际是否采用一一映射或直通。固件设置和平台文档说明机器具备什么能力，`/proc/cmdline` 显示传给内核的启动参数，内核启动日志可提供全局默认域的线索。查某个 PCI 功能时，再将 `/sys/bus/pci/devices/<BDF>/iommu_group` 对到实际组。在提供 `type` 文件的系统上，`/sys/kernel/iommu_groups/<组号>/type` 可读出组的默认域类型，例如 `identity` 或 `DMA`。要将设备、域和实际使用方式一起核对，组号只用来关联设备。虚拟化环境需要按对应平台的支持条件判断。Grace Blackwell 搭配 ConnectX-8 的 Data Direct 是另一条有独立软件和固件条件的设备路径。 [CUDA 13.1.2 · Supported Systems](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems) · [Linux v6.12 · IOMMU group sysfs ABI](https://github.com/torvalds/linux/blob/v6.12/Documentation/ABI/testing/sysfs-kernel-iommu_groups) · [NVIDIA · Grace Blackwell Data Direct 平台要求](https://docs.nvidia.com/multi-node-nvlink-systems/grace-blackwell-cx8-gpudirect-rdma-guide/platform_software_and_configuration.html)

## 两条注册链的前提不同

**DMA-BUF** 将一段缓冲区表示为一个可在本机驱动之间交接的文件描述符。先用 `CU_DEVICE_ATTRIBUTE_DMA_BUF_SUPPORTED` 查询 CUDA 设备能力，再用 `cuMemGetHandleForAddressRange()` 为符合条件的设备分配导出 fd。按 CUDA 13.1.2 的接口，地址和长度须按主机页大小对齐。例如普通 `cuMemAlloc()` 分配可走该接口，采用虚拟地址预留时还须核对整段已映射且满足接口的物理分配条件。取得 fd 之后，RDMA provider 才能用 `ibv_reg_dmabuf_mr()` 把该范围注册到指定保护域。导入参数包括 `fd`、`offset`、`length`、`iova` 和访问标志。`iova` 是通过 MR 的密钥访问时使用的虚拟基址，其页内偏移须和 `offset` 相同。该 fd 服务于本机导入，跨机交付给对端的是它执行操作所需的远程 MR 地址、范围和 `rkey`。 [CUDA 13.1.2 · `cuMemGetHandleForAddressRange`](https://docs.nvidia.com/cuda/archive/13.1.2/cuda-driver-api/group__CUDA__MEM.html) · [rdma-core v57.0 · ibv_reg_dmabuf_mr](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/man/ibv_reg_mr.3)

**Peer Memory** 通过 NVIDIA GPU 驱动提供的 `nvidia-peermem` 内核模块，让具备相应 peer-memory 接口的 RDMA 栈在处理 GPU 缓冲区注册时取得映射。检查已有机器时，把运行模块、NIC 驱动来源和实际注册路径对上。若只有 `nvidia_peermem` 的加载记录，只能确认模块已加载。老系统上的 `nv_peer_mem` 是较早的独立模块。CUDA 指南也指出，GPU 驱动先于 MLNX_OFED 安装时要确认它是否针对正确的 RDMA API 构建。 [CUDA 13.1.2 · Using nvidia-peermem](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#using-nvidia-peermem)

GPU Operator 25.3.4 的部署文档给出了下列共同前提。它们用于筛选一套候选组合，具体发行版、设备、驱动与固件仍要查目标平台的支持矩阵。 [GPU Operator 25.3.4 · Common Prerequisites](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/25.3.4/gpu-operator-rdma.html#common-prerequisites) · [GPU Operator 25.3.4 · Platform Support](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/25.3.4/platform-support.html)

| 核对项 | DMA-BUF | `nvidia-peermem` |
| --- | --- | --- |
| GPU 与分配类型 | 该文档列 Turing 或更新的相应数据中心 GPU、Quadro RTX 和 RTX GPU。现场还要查该分配能否导出。 | 使用该组合支持的 GPU 和可注册的 GPU 分配。 |
| GPU 内核模块 | 使用 NVIDIA 开放内核模块。 | 使用部署组合所支持的 GPU 驱动，并确认 `nvidia-peermem` 及其依赖。 |
| Linux 内核 | 该表列 Linux `5.12` 及以上。 | 查实际内核和 RDMA 栈是否具备所需 peer-memory 接口。 |
| NIC 驱动及固件 | 该文档允许使用支持该功能的发行版网络驱动，也可使用 MLNX_OFED 或 DOCA-OFED。NIC 与运行固件须支持对应导入。 | 该部署文档列 MLNX_OFED 或 DOCA-OFED。底层需要匹配的 peer-memory API。 |
| 用户态库与 provider | 该表列 CUDA `11.7` 及以上。还需有支持 DMA-BUF MR 的 Verbs 库和设备 provider。 | 核对库、provider 与内核驱动的实际组合，以及应用所走的显存注册路径。 |
| 测试程序构建 | 查 `--help` 和构建信息，确认程序能选 CUDA 缓冲区和 DMA-BUF 注册。 | 确认测试程序选用了 GPU 缓冲区及对应注册方式。 |

把四种证据分别记录。模块加载说明组件进入内核，CUDA 能导出 fd 说明这段分配完成了导出，MR 注册成功说明该 NIC 接受了这段映射和权限，传输完成才覆盖所测方向及操作。若现场停在注册阶段，先分清失败的调用发生在 CUDA 导出前后，再决定去查分配属性、provider、NIC 导入还是 PCIe 路径。
