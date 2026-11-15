
一项图像预处理任务需要解码一批压缩文件，再对每个像素做相同的数值变换。CPU 运行 Linux 和应用，负责读取文件、解码和组织输入。**GPU（Graphics Processing Unit，图形处理器）** 在这里充当并行计算加速器。各像素的变换可以独立执行，GPU 能把这批工作分给许多执行单元。应用通过 NVIDIA 的 **CUDA** 编程接口选择设备、安排复制并提交计算。驱动在应用与硬件之间建立访问和执行所需的状态。独立 GPU 有自己的显存，准备好的输入要先到达设备能读取的位置。本例把输入复制到 GPU 0 显存，结果也先写回这张卡。 [CUDA 13.2.1 · Programming Model](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/01-introduction/programming-model.html)

## 一批输入从盘到显存

采用一台 Linux 教学机，GPU 0 和 GPU 1 都有独立显存。本地文件放在 **NVMe（Non-Volatile Memory Express）** 固态盘上。这里的 NVMe 控制器经 **PCIe（Peripheral Component Interconnect Express）** 接入主机。同一主机也用 PCIe 接显卡和网卡。**PCIe Switch** 可从一条上行引出多条下行，把设备的请求和数据转发到对应端口。

1. **文件字节进入主机内存。** 应用发起普通缓冲式读取。假设文件数据尚未进入页缓存，Linux 文件系统和存储驱动要安排一次盘读。NVMe 控制器把文件字节写到主机内存中的页缓存，应用的普通 `read` 再从缓存取得文件字节。 [Linux 内核 · Page Cache](https://docs.kernel.org/mm/page_cache.html)
2. **CPU 产出可运算的数组。** 应用线程解码压缩文件，把像素值按 GPU 计算所需的布局写进另一个主机缓冲区。此时磁盘上的文件、页缓存里的文件字节和解码后的数组各有自己的用途。
3. **数组进入显存，GPU 执行变换。** 应用安排主机到设备的复制，驱动为设备建立访问缓冲区所需的映射。GPU 的搬运引擎把数组送入显存，随后 GPU 执行依赖该数组的计算。下一次 GPU 计算可直接用显存里的结果，主机若要保存结果，再安排一次显存到主机的复制。 [CUDA 13.2.1 · Unified and System Memory](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/02-basics/understanding-memory.html)

**DMA（Direct Memory Access，直接内存访问）** 让设备按驱动准备的地址读写内存，例如 NVMe 控制器把盘读结果写入主机内存、GPU 搬运引擎从主机取数组。CPU 执行应用和驱动代码，发起操作并在需要时等待完成。读文件的数据复制和图像解码在这条普通路径上仍有 CPU 工作。若输入放在普通可分页的内存中，CUDA 还可能先把它复制到可供设备访问的页锁定缓冲区。 [Linux 内核 · DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html) · [CUDA 13.2.1 · API 同步行为](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/api-sync-behavior.html)

![NVMe 控制器、网卡和两块 GPU 接到 PCIe Switch。橙色载荷从 NVMe 进入主机内存，再到 GPU 0 显存。紫色虚线从 CPU 指向 GPU 0，表示提交工作。](images/fig-01-01-01-device-path.png)

**图 01-01-01**　橙线画出两段载荷的方向，主机内存是前一段的落点和后一段的起点。图中的 CPU 方框合并了根端口和内存通路。GPU 0、GPU 1 各有 80 GB 标称显存，图中画的是教学机的安装形态。

文件命中页缓存时，应用可从主机内存取到文件字节，省去本次盘读。若输入改为远端存储，入口换成 **NIC（Network Interface Card，网络接口卡）**。NIC 把收到的数据写入主机侧的接收缓冲区，主机网络栈和存储客户端处理协议，应用再取得文件字节。两种变体都从应用拿到文件字节后进入同一条解码、复制和计算路径。第 06、09 章再分别讨论网卡和存储直接访问显存的条件。 [Linux 内核 · Page Cache](https://docs.kernel.org/mm/page_cache.html) · [Linux 内核 · DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html)

## 用 DGX H100 配置练习核算显存

工程上还需要知道每个阶段能使用多少资源。以下规格来自 DGX H100 整机手册，提供另一台机器的真实组件清单。 [NVIDIA DGX H100/H200 · 组件规格](https://docs.nvidia.com/dgx/dgxh100-user-guide/introduction-to-dgxh100.html#dgx-h100-h200-component-descriptions)

| 部件 | DGX H100 规格摘录 | 和本例的关系 |
| --- | --- | --- |
| CPU 与主机内存 | 2 颗 Xeon 8480C，每颗 56 核。32 条内存条，共 2 TB | 运行 Linux、解码文件，提供页缓存和应用缓冲区 |
| GPU 与显存 | 8 块 H100，每块 80 GB | 按实际执行的卡预算输入、输出和临时缓冲区 |
| NVMe 盘 | 2 × 1.92 TB M.2 系统盘，RAID 1 镜像。8 × 3.84 TB U.2 数据缓存盘，RAID 0 条带 | 保存系统和文件。可用盘容量取决于阵列组织 |
| ConnectX-7 网卡 | 8 块单端口集群卡。存储与带内管理有 2 块双端口卡 | 根据部署的地址、端口模式和接线找到远端输入的入口 |

八块卡的标称显存可作整机库存相加，得到 `8 × 80 GB = 640 GB`。普通设备内存分配按目标 GPU 结算。假设 GPU 0 和 GPU 1 各有 70 GiB 空闲，在 GPU 0 上申请 75 GiB，预算差 5 GiB。若算法允许按输入切成两个互不依赖的分片，每张卡分得 37.5 GiB，并各需 8 GiB 新增临时空间，则每张卡的新增峰值为 45.5 GiB，按采样各余 24.5 GiB。这个比较使用同一次采样和相同单位，其他已占用显存已计入空闲量。GB 保留厂商规格口径，GiB 按 2³⁰ 字节计。 [NVIDIA HGX H100 · 逐卡与整节点规格](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory-h100-h200-b200/latest/components.html) · [CUDA 13.2.1 · Multi-GPU Systems](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/03-advanced/multi-gpu-systems.html)

图中的“主机内存”在双路服务器上可能分布在两颗处理器旁边。要判断 GPU 0 取数组时是否跨过处理器互联，还得查设备接入点和这批数组实际占用的内存页。
