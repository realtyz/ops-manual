# GPU 间怎样获得直达访问

两张 GPU 接到同一台 PCIe Switch，拓扑图给出了一条可能的直达路径。要判断应用能否用它，还需检查**设备对、访问方向、启用状态及完成校验**。物理互联可以是 PCIe，也可以是 NVLink，CUDA 用 P2P 能力和权限描述特定设备对能否访问对方的显存。 [CUDA 13.2.1 · Programming Systems with Multiple GPUs](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/03-advanced/multi-gpu-systems.html)

## 先分清复制与访问方向

跨设备复制有源缓冲区和目的缓冲区。例如 `cudaMemcpyPeerAsync` 的参数分别指定源与目的设备，完成后目标显存持有一份数据。另一种操作是 Kernel 直接读写对方的显存。假设 GPU 0 的 Kernel 读取 GPU 1 用普通 `cudaMalloc` 分配的缓冲区，GPU 0 是访问者，GPU 1 是内存所有者。读取请求从 GPU 0 发向 GPU 1，响应的数据由 GPU 1 返回 GPU 0。 [CUDA 13.2.1 · Memory Management](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__MEMORY.html) · [Peer Device Memory Access](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__PEER.html)

对于这次远端访问，先检查 `cudaDeviceCanAccessPeer(&can, 0, 1)` 的返回码。调用成功时再读 `can`，`1` 表示 CUDA 报告设备 0 能访问设备 1。应用选中设备 0 后调用 `cudaDeviceEnablePeerAccess(1, 0)`，成功后建立这一方向的访问权限。若也要由 GPU 1 访问 GPU 0，需单独查询并从 GPU 1 侧启用。这里的设备序号属于应用进程，应先与实际 GPU 标识对齐。CUDA 指南也说明，适用的设备间复制在启用 P2P 后可以省掉主机暂存。仅凭 `cudaMemcpyPeerAsync` 出现在代码里，无法判读该次运行的复制实现及路径。 [CUDA 13.2.1 · Peer Device Memory Access](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__PEER.html) · [Multi-Device Peer-to-Peer Transfers and Memory Access](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/03-advanced/multi-gpu-systems.html#multi-device-peer-to-peer-transfers-and-memory-access)

## 查清平台能否承载这条路径

在四卡教学节点上，GPU 0 和 GPU 1 挂在 Switch A 下。如果 PCIe P2P 事务能在 Switch 内部转发，载荷从一条下行转到另一条下行，避开上行。若桥的 PCIe Access Control Services（ACS）对 P2P 请求或完成包启用了重定向，事务可能绕到根侧。**走根侧**描述事务经过的组件。**主机中转**描述有效载荷先写入主机 DRAM，再从该缓冲区发起第二次搬运。两者要分别验证。 [NCCL 2.31.2 · GPU Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/troubleshooting/gpu_troubleshooting.html)

IOMMU 可以对设备 DMA 地址做转换和隔离。CUDA 13.2.1 指南说明 CUDA 与 NVIDIA 驱动不支持在启用 IOMMU 的 Linux 裸机上做 PCIe P2P 内存传输，并将通过 VFIO 的虚拟机直通列为不同场景。即使工具报告 P2P 能力，也需结合本机实际启用的模式核对支持性。下面的检查用于采集证据，范围限于本例的 Linux 裸机 PCIe 路径。 [CUDA 13.2.1 · Host IOMMU Hardware, PCI Access Control Services, and VMs](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/03-advanced/multi-gpu-systems.html#host-iommu-hardware-pci-access-control-services-and-vms) · [NCCL 2.31.2 · GPU Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/troubleshooting/gpu_troubleshooting.html)

| 材料 | 重点看什么 | 判断到哪一步 |
| --- | --- | --- |
| `nvidia-smi topo -m`、PCIe 桥树 | 是否同一 Switch、是否经过根侧或跨 NUMA | 找候选路径与应检查的桥 |
| `nvidia-smi topo -p2p p` | 目标设备对的 PCIe P2P 状态 | 了解驱动报告的能力 |
| `lspci -D -s 0000:40:01.0 -vv` 等实际路径上的桥 | `ACSCtl` 中请求与完成包重定向，如 `ReqRedir`、`CmpltRedir` | 判断桥的策略是否可能让事务绕到根侧 |
| `/proc/cmdline` 和内核启动日志 | IOMMU 的启动参数及 `IOMMU`、`DMAR`、`Default domain` 等相关记录 | 用日志核对生效状态，启动参数只反映配置意图的一部分 |

## 用结果验证一次传输

下面是四卡 Linux 裸机教学节点的人工记录，按 CUDA 13.2.1 API 语义整理。符号 `A → B` 表示 **A 发起对 B 显存的访问**。题设已核实 GPU 0 与 GPU 1 的 PCIe 事务在 Switch A 内转发。GPU 0 与 GPU 2 的能力结论只针对这个教学节点。字段为教学汇总。

| 访问者 → 内存所有者 | 查询返回码 | `can` | 启用结果 | 测试完成 | 目标数据校验 |
| --- | --- | ---: | --- | --- | --- |
| GPU 0 → GPU 1 | `cudaSuccess` | 1 | `cudaSuccess` | 成功 | 通过 |
| GPU 1 → GPU 0 | `cudaSuccess` | 1 | `cudaSuccess` | 成功 | 通过 |
| GPU 0 → GPU 2 | `cudaSuccess` | 0 | 未调用 | 未执行 | 未执行 |
| GPU 2 → GPU 0 | `cudaSuccess` | 0 | 未调用 | 未执行 | 未执行 |

对于表中的 GPU 0 和 GPU 2，应用可显式安排 `GPU 0 → M0 → GPU 2`。分别等待 D2H 完成和 H2D 完成，再校验 GPU 2 的目标缓冲。这个结果验证的是中转实现。对于 GPU 0 与 GPU 1，若测试使用必须访问远端显存的 Kernel，并核对同步结果与数据，就能验证这对设备在该方向实际完成了远端访问。若还要归因到具体链路，结合测试的实现、链路状态和相应计数。测试吞吐只代表其设备、消息和实现条件。 [NVIDIA SMI · Topology](https://docs.nvidia.com/deploy/nvidia-smi/) · [NCCL 2.31.2 · GPU-to-GPU Communication](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/troubleshooting/gpu_troubleshooting.html#gpu-to-gpu-communication)
