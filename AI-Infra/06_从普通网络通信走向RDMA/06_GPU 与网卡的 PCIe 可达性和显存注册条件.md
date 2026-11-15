
要让网卡直接读写显存，需要本机两设备之间有可用的 PCIe 路径，还需要 GPU 驱动与 RDMA 栈为该段显存建立网卡可用的 DMA 映射。以下检查针对裸机上的传统 PCIe GPUDirect RDMA。先确定实际采用哪一对设备，再判断路径和注册条件。 [CUDA 13.1.2 的平台与路径条件](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems)

## 从设备标识还原实际 PCIe 路径

本例在节点 A 选 GPU 0 和 RDMA 设备 `mlx5_0` 的端口 1。用 `nvidia-smi --query-gpu=index,uuid,pci.bus_id --format=csv` 取得 GPU 的 PCI 地址。应用里的 CUDA 序号受设备可见性设置影响，应核对应用实际选到的卡。沿 `/sys/class/infiniband/mlx5_0/device` 找网卡的 PCI 地址，再用 `lspci -t` 在树上找两个地址的共同上游。本例两端具有同形的树，下面以 A 为例，BDF 为构造值。 [NVIDIA SMI 的 GPU 查询和拓扑矩阵](https://docs.nvidia.com/deploy/nvidia-smi/) · [CUDA 13.1.2 的设备枚举接口](https://docs.nvidia.com/cuda/archive/13.1.2/cuda-driver-api/group__CUDA__DEVICE.html)

```text
PCIe Root Complex（省略中间桥）
└── PCIe Switch
    ├── 下游桥 0000:81:01.0
    │   └── GPU 0     0000:82:00.0
    └── 下游桥 0000:81:02.0
        └── mlx5_0    0000:83:00.0（取端口 1）
```

`nvidia-smi topo -m` 中，`PIX` 表示经过一个 PCIe Switch，`PXB` 对应多个 PCIe Switch 且未过主桥，`PHB` 经过 PCIe Host Bridge，`SYS` 还经过处理器的 NUMA（Non-Uniform Memory Access，非一致内存访问）节点互联。共享交换芯片为芯片内转发提供了候选路径。经过主桥时，要核对平台能否将请求转回目标所在分支。传统路径的重要前提是两设备共享上游 PCIe Root Complex，跨 Socket 或跨根的连接要以目标平台明确支持的路径为准。同一个 NUMA 编号仍可能涉及多个 PCIe 分支。 [NVIDIA SMI 的 topo 图例](https://docs.nvidia.com/deploy/nvidia-smi/) · [CUDA 13.1.2 的 Supported Systems](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems)

## ACS 控制事务转发，IOMMU 管理 DMA 地址

**ACS（Access Control Services，访问控制服务）** 是 PCIe 层的访问控制能力。桥上的请求重定向 `ReqRedir` 和完成重定向 `CmpltRedir` 可以影响对等事务是否被发往上游。以 A 端网卡读取 GPU 为例，读请求由网卡发往 GPU，携带读回数据的 PCIe 完成再返回网卡。主题图中的载荷箭头对应后一方向。用 `lspci -vv -s 0000:81:02.0` 查看相应桥，`ACSCap` 是能力，`ACSCtl` 是控制位的读数。下面两行仅改变 A 的网卡侧下游桥上的一个控制位。 [Linux 内核 PCI Peer-to-Peer DMA](https://docs.kernel.org/driver-api/pci/p2pdma.html) · [pciutils v3.13.0 的 ACS 字段](https://github.com/pciutils/pciutils/blob/v3.13.0/ls-ecaps.c)

| 同一对设备的独立教学读数 | 拓扑距离 | 网卡侧下游桥的控制位 | 判断 |
| --- | --- | --- | --- |
| 条件甲 | `PIX` | `SrcValid+ ReqRedir- CmpltRedir-` | 这两个重定向位在该桥上关闭。继续核对另一侧及其余相关桥。 |
| 条件乙 | `PIX` | `SrcValid+ ReqRedir+ CmpltRedir-` | 网卡发起的对等请求可能被送上游。沿实际路径检查上游能否转发回目标。 |

部分裸机 RDMA 计算网络会按服务器平台指南，在节点内关闭 GPU 与网卡路径上相关 PCIe 桥的 ACS。若平台支持单独控制，也可只关闭相应的对等重定向位。以本例为例，当路径上的桥允许在同一 PCIe Switch 内转发时，网卡的读请求和返回数据都可以留在交换芯片内，省去经 Root Complex 折返的路径。对于依赖 ACS 实现设备隔离的虚拟化或设备直通部署，应保留平台要求的 ACS 设置。 [NVIDIA NCCL 2.27.3 的 ACS 排查说明](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html#pci-access-control-services-acs) · [Linux 内核 VFIO 的设备隔离说明](https://docs.kernel.org/driver-api/vfio.html)

**IOMMU（Input–Output Memory Management Unit，输入输出内存管理单元）** 为设备 DMA 提供地址转换和隔离。CUDA 13.1.2 所述的传统路径要求参与设备看到一致的物理地址，使用 IOMMU 时须核对该路径能否取得一一映射。固件设置、`/proc/cmdline` 和启动日志可提供平台与默认设置的线索。查某个 PCI 功能时，可由 `/sys/bus/pci/devices/<BDF>/iommu_group` 对到实际组，若该内核公开 `/sys/kernel/iommu_groups/<组号>/type`，再读其默认域类型，例如 `identity` 或 `DMA`。组号用于关联设备，默认域只说明默认映射方式。虚拟化和 Grace Blackwell 搭配 ConnectX-8 Data Direct 等路径需看各自的平台条件。 [CUDA 13.1.2 的 IOMMU 限定](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems) · [Linux v6.12 的 IOMMU group ABI](https://github.com/torvalds/linux/blob/v6.12/Documentation/ABI/testing/sysfs-kernel-iommu_groups) · [NVIDIA Grace Blackwell Data Direct 平台要求](https://docs.nvidia.com/multi-node-nvlink-systems/grace-blackwell-cx8-gpudirect-rdma-guide/platform_software_and_configuration.html)

## DMA-BUF 从本机导出到 MR 导入

1. **CUDA 分配和导出。** CUDA 13.1.2 可用 `CU_DEVICE_ATTRIBUTE_DMA_BUF_SUPPORTED` 检查设备内存的导出能力。对满足条件的 `cuMemAlloc()` 分配，`cuMemGetHandleForAddressRange()` 可导出 DMA-BUF 文件描述符。地址和长度须按主机页大小对齐。若使用预留虚拟地址，所选范围还须完全映射，所有背后的物理分配应位于同一设备且具有相同分配属性。 [CUDA 13.1.2 的显存导出接口](https://docs.nvidia.com/cuda/archive/13.1.2/cuda-driver-api/group__CUDA__MEM.html)
2. **RDMA 驱动导入并注册。** 应用用 `ibv_reg_dmabuf_mr()` 将文件描述符指定的范围注册进本机 PD。它提供 `fd`、文件内偏移 `offset`、长度、访问标志和 `iova`。后者是 MR 被密钥引用时使用的基址，其页内偏移须与 `offset` 相同。注册成功后，文件描述符服务于本机导入，A 远程写 B 时使用 B 发布的 RDMA 地址、长度和 `rkey`。 [rdma-core v57.0 的 DMA-BUF MR 注册](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/man/ibv_reg_mr.3)
3. **通过请求检验映射。** 导出成功说明 CUDA 接受了该分配的导出。导入成功进一步说明所选 RDMA 设备接受了注册参数。还需一次成功的实际请求，才能把证据扩展到所测的设备配对、方向和操作。

Peer Memory 路径由 `nvidia-peermem` 模块向具备相应接口的 RDMA 栈提供 GPU 对等内存映射。旧系统上可能见到早期独立模块 `nv_peer_mem`。模块已加载只能确认这项内核组件在运行。还需核对 NIC 驱动及其接口、程序实际采用的注册路径和注册结果。CUDA 13.1.2 指南还提醒，先于 MLNX_OFED 安装 GPU 驱动时，应确保 GPU 驱动按目标 RDMA API 构建。 [CUDA 13.1.2 的 nvidia-peermem 部署与兼容说明](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#using-nvidia-peermem)

GPU Operator 25.3.4 的部署文档将 DMA-BUF 路径的前提列为 NVIDIA 开放内核模块、CUDA 11.7 或更高、Linux 5.12 或更高，以及 Turing 或更新的相应 GPU，可用受支持的发行版网络驱动或厂商驱动。它为 `nvidia-peermem` 路径列出 MLNX_OFED 或 DOCA-OFED。使用这些前提筛选候选组合后，还要核对目标平台、实际 GPU 分配、NIC、运行固件以及用户态 provider。 [GPU Operator 25.3.4 的 RDMA 部署前提](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/25.3.4/gpu-operator-rdma.html#common-prerequisites) · [GPU Operator 25.3.4 的平台矩阵](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/25.3.4/platform-support.html)

若设备和拓扑均匹配，先跑主机内存 RDMA，再换成显存缓冲区。分别记录 CUDA 导出、RDMA 导入和工作请求完成，就能把失败定位到比“GPUDirect 不工作”更具体的阶段。
