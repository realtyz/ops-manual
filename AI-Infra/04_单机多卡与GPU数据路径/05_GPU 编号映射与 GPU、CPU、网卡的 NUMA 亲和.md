
进程日志写 `cuda:0`，主机拓扑矩阵写 `GPU2`。画这次传输的路径，先要知道进程把数据交给了哪张卡，再沿桥树找到接入位置。GPU 接在哪个 NUMA 节点提供了就近放置的线索。判断 DMA 是否经过处理器间互联，还要找到缓冲区的物理页。下面回到双路四卡教学节点，标识、字段和采样值均为教学材料。

## CUDA 进程序号与物理 GPU 标识

CUDA 给可见 GPU 编连续的进程内序号（ordinal）。当应用由 `CUDA_VISIBLE_DEVICES` 限制可见范围并指定顺序时，序号 0 指列表第一张卡。主机的 `nvidia-smi` index 按驱动的枚举顺序给出。要把两套编号对上，可以用 GPU UUID（通用唯一标识符）确认是哪张设备，再用完整 PCI BDF 找它本次枚举的位置。以下为 `nvidia-smi --query-gpu=index,uuid,pci.bus_id --format=csv` 的教学整理。例中的四位域号 `0000` 与工具显示的八位 `00000000` 表示同一个 PCI 域。重启或调整硬件后用同轮采集重新对照。 [NVIDIA SMI · 设备标识](https://docs.nvidia.com/deploy/nvidia-smi/) · [CUDA 13.2.1 · 环境变量](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html)

| 主机 index | GPU UUID | PCI BDF |
| ---: | --- | --- |
| 0 | `GPU-11111111-1111-1111-1111-111111111111` | `0000:41:00.0` |
| 1 | `GPU-22222222-2222-2222-2222-222222222222` | `0000:42:00.0` |
| 2 | `GPU-33333333-3333-3333-3333-333333333333` | `0000:c1:00.0` |
| 3 | `GPU-44444444-4444-4444-4444-444444444444` | `0000:c2:00.0` |

启动应用前按下面的顺序列 GPU 2 和 GPU 0。进程内对设备 0 调用 `cudaDeviceGetPCIBusId`，本例返回 `0000:c1:00.0`，查表对应主机 GPU 2。进程内设备 1 返回 `0000:41:00.0`，对应主机 GPU 0。分析使用进程序号的方向记录时，先做这一步转换。 [CUDA 13.2.1 · Runtime 设备管理](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)

```bash
# 教学配置。运行时用目标主机查到的 UUID
export CUDA_VISIBLE_DEVICES=GPU-33333333-3333-3333-3333-333333333333,GPU-11111111-1111-1111-1111-111111111111
```

## 拓扑矩阵的距离类别与真实桥路径

以下为 `nvidia-smi topo -m` 的教学节选，假设两个物理网卡均列在矩阵里。GPU 2 行显示它到 GPU 3、NIC 1 均为 `PIX`，到 GPU 0、NIC 0 均为 `SYS`。右侧亲和列把 GPU 2 指向逻辑 CPU 32–63 和 NUMA 1。部分平台或工具版本会显示额外的 GPU NUMA ID，按实际表头和工具图例解读。 [NVIDIA SMI · Topology](https://docs.nvidia.com/deploy/nvidia-smi/)

```text
       GPU0 GPU1 GPU2 GPU3 NIC0 NIC1 CPU Affinity NUMA Affinity
GPU0   X    PIX  SYS  SYS  PIX  SYS  0-31         0
GPU1   PIX  X    SYS  SYS  PIX  SYS  0-31         0
GPU2   SYS  SYS  X    PIX  SYS  PIX  32-63        1
GPU3   SYS  SYS  PIX  X    SYS  PIX  32-63        1
```

| 标记 | 工具报告的连接类别 |
| --- | --- |
| `PIX` | 经过一个 PCIe Switch |
| `PXB` | 经过多个 PCIe Switch，未经过 PCIe Host Bridge |
| `PHB` | 经过 PCIe Host Bridge |
| `NODE` | 经 PCIe 和同一 NUMA 节点内 PCIe Host Bridge 之间的互联 |
| `SYS` | 经 PCIe 和 NUMA 节点间的系统互联 |
| `NV#` | 经过由 `#` 条 NVLink 组成的连接 |

若有 NVLink，可用 `nvidia-smi topo -mp` 单看 PCIe。距离类别缩小了查找范围，具体公共链路还要用 `lspci -D -t` 按 BDF 回溯。以下是同一节点桥树的整理。GPU 2 的地址落在 Switch B 下，GPU 0 落在 Switch A 下。从各自的本地内存读取时，两卡载荷分别使用 B 和 A 的上行。 [NVIDIA SMI · Topology](https://docs.nvidia.com/deploy/nvidia-smi/) · [pciutils · lspci 手册](https://github.com/pciutils/pciutils/blob/v3.13.0/lspci.man)

| 根端口与 Switch 上行端口 | Switch 下行端点 |
| --- | --- |
| Socket 0，`0000:20:01.0` → `0000:3f:00.0`（A） | GPU 0 `0000:41:00.0`，GPU 1 `0000:42:00.0`，NIC 0 `0000:43:00.0` |
| Socket 1，`0000:80:01.0` → `0000:bf:00.0`（B） | GPU 2 `0000:c1:00.0`，GPU 3 `0000:c2:00.0`，NIC 1 `0000:c3:00.0` |

把应用使用的物理网口也对回 BDF。例如读取 `/sys/class/net/enp195s0f0/device` 符号链接，教学节点上其末端是 `0000:c3:00.0`，对应 NIC 1。相对 NIC 0，NIC 1 与 GPU 2 共用 Switch B，存在避开根侧和处理器间互联的候选路径。多口网卡还要保留端口到接口的对应关系。网卡能否使用 GPUDirect RDMA 访问显存，仍需核对设备、驱动、内存注册及实际传输，第 06 章展开这条路径。 [Linux · Sysfs](https://docs.kernel.org/filesystems/sysfs.html) · [NVIDIA · GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/index.html)

## CPU 亲和与缓冲区实际驻留页

| 要确认的位置 | Linux 观察点 |
| --- | --- |
| PCI 设备的亲和节点 | `/sys/bus/pci/devices/<BDF>/numa_node`。值为 `-1` 时表示未知，可结合桥树和平台资料补证。 |
| 节点包含的逻辑 CPU | `/sys/devices/system/node/node<N>/cpulist`。 |
| 线程的 CPU 和内存允许范围 | `/proc/<pid>/task/<tid>/status` 的 `Cpus_allowed_list` 和 `Mems_allowed_list`。允许范围用于理解进程能在哪运行或分配。 |
| 线程最近执行的 CPU | `/proc/<pid>/task/<tid>/stat` 的 `processor` 字段。 |
| 缓冲区的实际页位置 | 用 `/proc/<pid>/maps` 找包含地址的映射范围，以映射起始地址对到 `numa_maps`。其中 `N0`、`N1` 等为该映射在各节点的页数。 |

应用将输入放在独立的匿名映射，起始地址为 `0x7f0000000000`。假设匹配的 `numa_maps` 行为 `7f0000000000 default anon=256 N0=256 kernelpagesize_kB=4`，则这次采集记录了 M0 上的 256 个 4 KiB 页，合计 1 MiB。假设这些页已锁定，DMA 直接读取这些页，采样后到传输时页位置未变。给 `cuda:0` 搬运该缓冲区时，载荷从 M0 经处理器互联到 Socket 1，再经上行 B 到 GPU 2，即使提交线程的 `Cpus_allowed_list` 是 `32-63`。若目标缓冲与其他对象共用同一映射，`numa_maps` 给的是整段统计。新页的位置还会受首次触页、内存策略和允许节点影响，观察应对准参与该次 DMA 的缓冲区和采样时刻。 [Linux · PCI sysfs ABI](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci) · [NUMA 内存策略](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html) · [proc 文件系统](https://docs.kernel.org/filesystems/proc.html)

这次改动给上行 B 保留了独立性，但在上游增加了一段处理器互联。比较不同 GPU 放置时，需要把实际内存路径和卡间交换路径一起计入每轮时间。
