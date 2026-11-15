
一批输入已经放进主机内存，GPU 上的计算要等输入进入自己的显存。CPU 线程调用 CUDA 复制接口，CUDA Runtime 和驱动安排设备工作。本例由 GPU 的复制引擎搬运。**DMA（Direct Memory Access，直接内存访问）**是设备按已建立的地址映射访问内存的机制。CPU 负责准备数据、提交和检查结果，复制引擎负责逐块搬运有效载荷。 [CUDA 13.2.1 · 系统内存与显存](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/02-basics/understanding-memory.html) · [异步执行](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/02-basics/asynchronous-execution.html)

## 本地 H2D 的读取请求、回包与完成

本章用一台双路 x86_64 Linux 裸机作教学节点。NUMA（Non-Uniform Memory Access，非一致内存访问）描述 CPU 访问本地与远端内存的代价差异。本例每个 CPU 插座（Socket）对应一个 NUMA 节点，包含该侧的 CPU 与本地主机内存。Socket 0 的本地内存记为 M0，Socket 1 的记为 M1。两颗处理器通过彼此之间的互联访问对方的内存。

| 接入位置 | 从处理器侧到设备的连接 |
| --- | --- |
| Socket 0 / NUMA 0 | PCIe 根端口 → Switch A → GPU 0、GPU 1、NIC 0 |
| Socket 1 / NUMA 1 | PCIe 根端口 → Switch B → GPU 2、GPU 3、NIC 1 |

根端口从处理器侧引出 PCIe 分支。交换机朝根端口的连接称为**上行**，朝各 GPU 或网卡（NIC，Network Interface Card）的连接称为**下行**。A、B 各有独立上行。四张 GPU 各有显存，彼此之间没有 NVLink 连接。链路两端可双向传输，上行和下行是端口在拓扑中的位置。

从 M0 向 GPU 0 发起一次 H2D（Host to Device，主机到设备拷贝）。假设源缓冲区位于 M0，应用已将其页锁定，使这些页在传输期间保持可供 DMA 访问。目的缓冲区已在 GPU 0 显存中分配。

1. CPU 线程把复制提交到 CUDA Stream。Stream 是一条有序的工作队列。复制排到可执行时，GPU 侧复制引擎向源地址发出 PCIe 读取请求。
2. 读取请求经 GPU 0 的下行和 Switch A 的上行抵达 Socket 0 根侧。返回的有效载荷沿反方向移动，路径为 **M0 → Socket 0 根侧 → Switch A → GPU 0 显存**。本章按有效载荷方向计算每段链路承担的字节数。
3. 复制完成后，同一 Stream 中排在它后面的 Kernel（GPU 上执行的程序）才会读到这批数据。主机若要复用源缓冲区，需等复制完成，例如检查复制所属 Stream 的同步结果。返回成功的提交调用和复制完成是两个时点。 [CUDA 13.2.1 · CUDA Stream 顺序与显式同步](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/02-basics/asynchronous-execution.html)

若改成把 GPU 0 的结果送回 M0，操作叫 D2H（Device to Host，设备到主机拷贝）。复制引擎把有效数据从显存经 Switch A 和 Socket 0 根侧写入 M0。主机在这次 D2H 完成后读取结果。

## 远端页和可分页缓冲区怎样改变 DMA 路径

固定目的地 GPU 0，把用于 DMA 的整个源缓冲区放到 M1。返回的有效数据先从 M1 经 Socket 1 和处理器间互联到达 Socket 0，再沿原来的 PCIe 分支进入 GPU 0。多出来的是处理器间互联这一段。提交线程的 CPU 亲和可能影响首次触页的位置，已有缓冲区的传输路径由其实际物理页位置决定。 [Linux · NUMA 内存策略](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html)

应用可用 `cudaMallocHost` 分配页锁定缓冲区，或在支持该操作的系统上用 `cudaHostRegister` 注册已有缓冲区。普通可分页内存可能需要经过页锁定暂存区。发生暂存时，先把字节从原缓冲区移到暂存区，随后的 DMA 源和 NUMA 路径要按暂存区所在的页追踪。 [CUDA 13.2.1 · 页锁定内存](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/02-basics/understanding-memory.html#page-locked-host-memory) · [Runtime 内存管理](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__MEMORY.html)

在这个 H2D 场景下，CUDA 13.2.1 对调用返回时点的约定如下。给复制计时时，要覆盖实际完成时点。

| 复制调用 | 调用返回与设备完成的关系 |
| --- | --- |
| `cudaMemcpy`，源为页锁定内存 | 对主机同步，检查成功返回即可确认本次拷贝完成。 |
| `cudaMemcpy`，源为可分页内存 | 复制前同步相应的 Stream。数据进了暂存区即可返回，最终 DMA 可能仍在进行。 |
| `cudaMemcpyAsync`，源为页锁定内存 | 对主机异步，检查提交结果后，用所属 Stream 或记录在复制后的事件确认完成。 |
| `cudaMemcpyAsync`，源为可分页内存 | 可能对主机同步。驱动若需暂存，可能同步 Stream 再搬到页锁定区。仍以相应 Stream 或事件判断完成。 |

异步提交给应用留下安排重叠的机会。实际能否并行，还要看设备的执行资源、缓冲区和 Stream 之间的依赖。 [CUDA 13.2.1 · API 同步行为](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/api-sync-behavior.html)

## GPU 间直达与主机内存中转

把源缓冲区改为 GPU 0 的显存，目的改为 GPU 1 的显存。两张卡有一条经 Switch A 的候选直达路径。**P2P（Peer-to-Peer，对等访问）**在这里指 GPU 可以经设备互联直接访问另一张卡的显存。若所需设备对和方向可用，应用使用直达传输，且 PCIe 事务在 Switch A 内转发，载荷便从 GPU 0 的下行转到 GPU 1 的下行。两张卡仍各自拥有自己的显存。 [CUDA 13.2.1 · 多 GPU 对等传输](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/03-advanced/multi-gpu-systems.html#multi-device-peer-to-peer-transfers-and-memory-access)

同一份数据也可以经 M0 显式中转。应用安排 GPU 0 的 D2H，将数据写入中转缓冲区。数据到达后，GPU 1 才能从同一缓冲区做 H2D。每个字节先经过上行 A 的 Switch→主机方向，再经过主机→Switch 方向。两段的先后依赖来自它们读写同一批字节。

![同一 PCIe 分支上的两种卡间路径。左图从 GPU 0 显存经 Switch A 到 GPU 1 显存。右图先以琥珀色 D2H 到 M0 缓冲区，再以紫色 H2D 到 GPU 1 显存。](images/fig-04-01-01-p2p-vs-host-staging.png)

**图 04-01-01**　四卡节点中 Switch A 的局部连接。左图假定 P2P 在 Switch 内转发，载荷避开上行和主机内存。右图中同色线段属于同一步，Socket 方框代表处理器侧的内存与 PCIe 通路，点线标记 CPU 提交。

这样可以直接从路径看出哪些流量会相遇。两张 GPU 同时从 M0 读输入时，两个载荷都流过上行 A 的主机→Switch 方向。若改成同分支直达交换，载荷占用两卡下行，上行预算便可留给经过它的其他数据。
