# 从拓扑输出还原设备和 NUMA 位置

应用日志写 `cuda:0`，运维材料写 `GPU2`。两者可能指同一张物理卡。先用 UUID 或完整 PCI BDF 对齐身份，再读拓扑矩阵和 PCIe 树。最后看缓冲区的实际页落点，判断这次传输要走哪条路径。 [NVIDIA SMI · Device Identification](https://docs.nvidia.com/deploy/nvidia-smi/) · [CUDA 13.2.1 · CUDA Environment Variables](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html)

## 从进程内序号找到主机设备

本例是双路 x86_64 Linux 裸机教学节点。每个 Socket 对应一个 NUMA 节点。GPU 0、GPU 1 与 NIC 0 在 Switch A 下，GPU 2、GPU 3 与 NIC 1 在 Switch B 下，两个 Switch 有独立上行。下文的设备标识、命令片段和内存页数均为教学示例。BDF 由域、总线、设备、功能四部分组成，表格统一将域号写为四位。工具若显示 `00000000:c1:00.0`，与 `0000:c1:00.0` 是同一个地址。

在主机上运行 `nvidia-smi --query-gpu=index,uuid,pci.bus_id --format=csv`。取同一轮采集的 UUID 与 BDF，连回该轮的主机 index。index 是管理工具的枚举序号，GPU UUID 标识设备，BDF 则标识本次系统枚举中的 PCI 功能。重启或改变设备位置后应重新采集。 [NVIDIA SMI · Query Options](https://docs.nvidia.com/deploy/nvidia-smi/)

| 本次主机 index | GPU UUID | PCI BDF | 分支 |
| ---: | --- | --- | --- |
| 0 | `GPU-11111111-1111-1111-1111-111111111111` | `0000:41:00.0` | A |
| 1 | `GPU-22222222-2222-2222-2222-222222222222` | `0000:42:00.0` | A |
| 2 | `GPU-33333333-3333-3333-3333-333333333333` | `0000:c1:00.0` | B |
| 3 | `GPU-44444444-4444-4444-4444-444444444444` | `0000:c2:00.0` | B |

现在在启动应用前设置 `CUDA_VISIBLE_DEVICES`，先列 GPU 2 的 UUID，再列 GPU 0 的 UUID。应用可见的设备顺序由这份列表决定。让应用记录每个 ordinal 调用 `cudaDeviceGetPCIBusId` 得到的 BDF，与主机表交叉验证。 [CUDA 13.2.1 · CUDA Environment Variables](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html) · [Runtime API · Device Management](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)

```bash
# 教学配置，UUID 换成目标主机实际查到的值
export CUDA_VISIBLE_DEVICES=GPU-33333333-3333-3333-3333-333333333333,GPU-11111111-1111-1111-1111-111111111111
```

| 应用 ordinal | 应用记录的 BDF | 对应主机 GPU |
| ---: | --- | --- |
| 0 | `0000:c1:00.0` | GPU 2 |
| 1 | `0000:41:00.0` | GPU 0 |

## 从矩阵缩小范围，再回到桥树

以下是人工整理的 `nvidia-smi topo -m` 节选，保留四张 GPU 的行。假设两个 NIC 都被这个版本的工具列入矩阵。`CPU Affinity` 与 `NUMA Affinity` 描述工具报告的 GPU 亲和关系。不同工具版本或平台还可能显示额外的 GPU NUMA ID 等字段，应按实际表头读取。 [NVIDIA SMI · Topology](https://docs.nvidia.com/deploy/nvidia-smi/)

```text
       GPU0 GPU1 GPU2 GPU3 NIC0 NIC1 CPU Affinity NUMA Affinity
GPU0   X    PIX  SYS  SYS  PIX  SYS  0-31         0
GPU1   PIX  X    SYS  SYS  PIX  SYS  0-31         0
GPU2   SYS  SYS  X    PIX  SYS  PIX  32-63        1
GPU3   SYS  SYS  PIX  X    SYS  PIX  32-63        1
```

| 标记 | `topo -m` 中表示的连接 |
| --- | --- |
| `PIX` | 经过一个 PCIe Switch |
| `PXB` | 经过多个 PCIe Switch，未经过 PCIe Host Bridge |
| `PHB` | 经过 PCIe Host Bridge |
| `NODE` | 经过 PCIe，并在同一 NUMA 节点内跨 PCIe Host Bridge 的互联 |
| `SYS` | 经过 PCIe 与 NUMA 节点间的系统互联 |
| `NV#` | 经过由 `#` 条 NVLink 组成的连接 |

矩阵给的是连接类别。`NV#` 不能说明某次复制已使用 NVLink，`SYS` 也不能单独给出吞吐或设备对的 P2P 支持。需要单看 PCIe 连接时，用 `nvidia-smi topo -mp`。再用 `lspci -D -t` 追桥，确定哪些端点真正共用一段上行。 [NVIDIA SMI · Topology](https://docs.nvidia.com/deploy/nvidia-smi/) · [pciutils · lspci 手册](https://github.com/pciutils/pciutils/blob/master/lspci.man)

| 教学节点的桥路径 | 各下行端点 |
| --- | --- |
| Root Port `0000:20:01.0` → Switch A 上行端口 `0000:3f:00.0` → 总线 `40` 下的三个下行端口 `01.0`、`02.0`、`03.0` | GPU 0 `0000:41:00.0`，GPU 1 `0000:42:00.0`，NIC 0 `0000:43:00.0` |
| Root Port `0000:80:01.0` → Switch B 上行端口 `0000:bf:00.0` → 总线 `c0` 下的三个下行端口 `01.0`、`02.0`、`03.0` | GPU 2 `0000:c1:00.0`，GPU 3 `0000:c2:00.0`，NIC 1 `0000:c3:00.0` |

网口名也要映射到 BDF。教学节点上，假设两个物理接口是 `enp67s0f0` 和 `enp195s0f0`。查询 `/sys/class/net/enp67s0f0/device` 与 `/sys/class/net/enp195s0f0/device` 的符号链接，路径末端分别是 `0000:43:00.0` 与 `0000:c3:00.0`。网口与 PCI 功能并非普遍一一对应，多口设备还应保留具体端口映射。若 GPU 2 与 NIC 1 是 `PIX`，与 NIC 0 是 `SYS`，NIC 1 就是本例拓扑上的就近候选。是否具备 GPUDirect RDMA，要另查 GPU、NIC、驱动、内存注册和传输证据。 [Linux · Sysfs](https://docs.kernel.org/filesystems/sysfs.html) · [NVIDIA · GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/index.html)

## 分清设备亲和、线程位置和缓冲区页落点

| 要核实的位置 | 查看什么 |
| --- | --- |
| GPU 或 NIC 的亲和节点 | `/sys/bus/pci/devices/<BDF>/numa_node`。`-1` 表示未知，应结合桥树和平台资料补证 |
| 某 NUMA 节点有哪些逻辑 CPU | `/sys/devices/system/node/node<N>/cpulist` |
| 目标线程允许运行在哪些 CPU | `/proc/<pid>/task/<tid>/status` 的 `Cpus_allowed_list`。`Mems_allowed_list` 是允许分配内存的节点范围 |
| 线程最近运行在哪个 CPU | `/proc/<pid>/task/<tid>/stat` 的 `processor` 字段给出最近执行的 CPU，是一次快照 |
| 源缓冲区的物理页实际在哪里 | 用应用记录的地址匹配 `/proc/<pid>/numa_maps` 对应的映射，读 `N0`、`N1` 等每节点页数。若映射混有其他缓冲区，整段统计无法精确归因给目标缓冲区 |

举例，应用把输入放到一个独立匿名映射，起始地址为 `0x7f0000000000`。设采样时匹配行是 `7f0000000000 default anon=256 N0=256 kernelpagesize_kB=4`，则这些 `256` 页在 NUMA 0，共 `1 MiB`。这时应用 ordinal 0 是 NUMA 1 的 GPU 2。假设源缓冲区已成功锁页、DMA 直接读取这些页，且采样到传输间页未迁移，H2D 就要跨 Socket。即使线程的 `Cpus_allowed_list` 已为 `32-63`，现有页仍在 M0。新页则可能受首次触页、显式内存策略和允许节点影响，后续迁页也会改变判断。 [Linux · PCI sysfs ABI](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci) · [NUMA Memory Policy](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html) · [The /proc Filesystem](https://docs.kernel.org/filesystems/proc.html)
