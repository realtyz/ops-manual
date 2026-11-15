
GPU 0 刚算出一份结果，GPU 1 要使用它。同一进程看到这两张卡。它们各有显存，同接 Switch A。直达的条件有两层。硬件必须能转发这对设备的事务，CUDA 也必须能让访问方使用相应的显存地址。从一次 Kernel 读取可以看清这两层如何配合。 [CUDA 13.2.1 · 多设备对等传输与内存访问](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/03-advanced/multi-gpu-systems.html#multi-device-peer-to-peer-transfers-and-memory-access)

## 对端显存读取的方向、权限和完成

假设单进程中设备序号 `0`、`1` 分别对应教学节点的 GPU 0、GPU 1。这个序号称为 ordinal，按进程的可见设备列表从零编号。GPU 0 用普通 `cudaMalloc` 分配一块显存并写入已知数据。GPU 1 要在自己的 Kernel 中读取 GPU 0 的指针，并把读到的值写入自己的一块结果缓冲。

1. 主机先调用 `cudaDeviceCanAccessPeer(&can, 1, 0)`。接口返回成功后，`can = 1` 表示 CUDA 报告设备 1 可以访问设备 0 的显存。返回码和能力输出各有含义。
2. 选中设备 1，再调用 `cudaDeviceEnablePeerAccess(0, 0)`。调用成功后，GPU 1 的 Kernel 可通过该设备指针访问 GPU 0 的这类分配。若还要让 GPU 0 访问 GPU 1，须从 GPU 0 一侧单独查询并启用。
3. 主例由主机先等待 GPU 0 的写入完成，再提交 GPU 1 的读取。访问请求从 GPU 1 发往 GPU 0，有效数据从 GPU 0 返回 GPU 1。等读取完成后，将 GPU 1 的结果拷回并校验。地址访问权限和生产者、消费者的执行顺序各由相应机制保证。 [CUDA 13.2.1 · Peer Device Memory Access](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__PEER.html) · [多设备 Stream 和内存一致性](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/03-advanced/multi-gpu-systems.html)

这一次的**访问方向是 GPU 1 → GPU 0，载荷方向是 GPU 0 → GPU 1**。如果应用希望在 GPU 1 留下一整块可反复使用的本地数据，也可以用设备间复制接口，例如 `cudaMemcpyPeerAsync`，传入源和目的设备及缓冲区。CUDA 指南说明，当对等访问已启用时，设备间复制可省去主机暂存。接口参数描述数据交付给谁，路由还取决于设备对的能力、互联和运行实现。 [CUDA 13.2.1 · 对等内存传输](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/03-advanced/multi-gpu-systems.html#peer-to-peer-memory-transfers) · [Runtime 内存管理](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__MEMORY.html)

## PCIe 交换机内转发与根侧重定向

若 GPU 0 和 GPU 1 的对等事务在 Switch A 内转发，请求和回包在两卡下行间交换，占用相应的方向带宽。当 PCIe 桥的 ACS（Access Control Services，访问控制服务）启用了相关的请求或完成包重定向，事务可能被送到根侧。此时上行会参与传输，能否继续抵达对端还要看平台支持。这里的载荷目的仍是 GPU 显存。主机中转则需要先把载荷写入主机内存，再发起下一段 H2D。两种路径对上行、主机内存的影响都可按载荷落点逐段推导。 [Linux 内核 · PCI Peer-to-Peer DMA](https://docs.kernel.org/driver-api/pci/p2pdma.html) · [NCCL 2.31.2 · PCI Access Control Services](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/troubleshooting/gpu_troubleshooting.html#pci-access-control-services-acs)

IOMMU（Input-Output Memory Management Unit，输入输出内存管理单元）负责设备 DMA 的地址转换和访问隔离。CUDA 13.2.1 指南说明，CUDA 与 NVIDIA 驱动在启用 IOMMU 的 Linux 裸机上不支持 PCIe P2P 内存传输。指南单列了通过 VFIO 做虚拟机 PCIe 直通的场景，其支持条件需按虚拟化配置另查。以下观察针对本章裸机的 GPU 间 PCIe P2P。 [CUDA 13.2.1 · IOMMU、ACS 和虚拟机](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/03-advanced/multi-gpu-systems.html#host-iommu-hardware-pci-access-control-services-and-vms)

| 查什么 | 能回答的问题 |
| --- | --- |
| `nvidia-smi topo -m` 和 `lspci -D -t` | 两张卡在哪些桥下，可能共用哪些物理段？ |
| `nvidia-smi topo -p2p p` | 驱动针对目标设备对报告什么 PCIe P2P 状态？ |
| 路径上的桥，例如 `lspci -D -s 0000:40:01.0 -vv` | `ACSCtl` 的 `ReqRedir`、`CmpltRedir` 是否提示相关事务可能被重定向？ |
| `/proc/cmdline` 和本次启动的内核日志 | 启动参数意图是什么，日志报告了怎样的 IOMMU 状态？ |

`topo -p2p p` 的结果适合与 CUDA 的有向查询交叉核对。观察到桥策略或 IOMMU 相关字段后，仍要以目标设备对的支持条件和功能验证判断这次访问。 [NVIDIA SMI · Topology](https://docs.nvidia.com/deploy/nvidia-smi/) · [pciutils · ACS 字段](https://github.com/pciutils/pciutils/blob/v3.13.0/ls-ecaps.c) · [NCCL 2.31.2 · GPU 排查](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/troubleshooting/gpu_troubleshooting.html)

## 设备对能力、数据校验与实际选路证据

下面是四卡教学节点的方向记录。箭头表示**访问者 → 内存所有者**，主例对应第一行。测试采用直接读对端指针的 Kernel。

| 访问方向 | 查询返回码 | `can` | 启用 | 完成与校验 |
| --- | --- | ---: | --- | --- |
| GPU 1 → GPU 0 | `cudaSuccess` | 1 | 成功 | 成功，数据正确 |
| GPU 0 → GPU 1 | `cudaSuccess` | 1 | 成功 | 成功，数据正确 |
| GPU 2 → GPU 0 | `cudaSuccess` | 0 | 未调用 | 未执行 |
| GPU 0 → GPU 2 | `cudaSuccess` | 0 | 未调用 | 未执行 |

第一行验证 GPU 1 在给定条件下读到了 GPU 0 的数据。要确认载荷一直留在 Switch A 内，还需结合互联、桥配置以及平台可用的链路流量观测。本章将交换机内转发作为该设备对的额外题设。GPU 0/2 的查询也成功返回，`can = 0` 给出本例的能力结论。若要把 GPU 0 的结果交付给 GPU 2，应用可以显式先 D2H 到 M0，再从 M0 经处理器互联和 Switch B 做 H2D，分别等待两段完成并校验目标。这里校验通过证明的是这条中转实现。 [NCCL 2.31.2 · GPU-to-GPU Communication](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/troubleshooting/gpu_troubleshooting.html#gpu-to-gpu-communication)

有 NVLink 的平台多了一种承载 P2P 的互联。相同的访问者和显存所有者仍可用于描述这次操作，路由和就绪条件则由另一组链路与管理组件决定。
