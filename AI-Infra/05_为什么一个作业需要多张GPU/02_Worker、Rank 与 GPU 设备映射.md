
设节点 A、B 各有四张 NVIDIA 整卡。在每个节点上，PyTorch 的进程启动器 `torchrun` 运行一个 **Agent**，负责启动和监控本机的训练进程。这些执行训练代码的 Linux 进程叫 **Worker**。本例每节点四个 Worker，每个 Worker 独占一张 GPU。要核对它们在哪些设备上协作，需要先分清进程编号和设备编号。 [PyTorch 2.9 · torchrun](https://docs.pytorch.org/docs/2.9/elastic/run.html)

## Worker 的全局和本地编号

**Rank** 是成员在某个组里的编号，从 0 开始。以下是 `torchrun` 传给 Worker 的环境变量，同一变量要在本轮运行和相应的范围内解释。A 上分到 0—3 是这次教学运行的设定。 [PyTorch 2.9 · torchrun 环境变量](https://docs.pytorch.org/docs/2.9/elastic/run.html#environment-variables)

| 变量 | 范围 | 本轮示例 |
| --- | --- | --- |
| `RANK` | 本轮整个 Worker 组内的编号 | A 为 0—3，B 为 4—7 |
| `WORLD_SIZE` | 本轮整个 Worker 组的成员数 | 8 |
| `LOCAL_RANK` | 本节点的本地 Worker 组内的编号 | A、B 各为 0—3 |
| `LOCAL_WORLD_SIZE` | 本地 Worker 组的成员数 | A、B 各有 4 个 |

A 上的全局 Rank 0 和 B 上的全局 Rank 4 都可以有 `LOCAL_RANK=0`。要定位 Linux 进程，本例还记录主机与 PID。重启会产生新 PID，重新组队也可能重新分配 Rank。采集时把运行 ID 和尝试编号一起记下。 [PyTorch 2.9 · torchrun 的编号与重启约束](https://docs.pytorch.org/docs/2.9/elastic/run.html#important-notices)

## 设备序号、UUID 和 PCI 位置

第 04 章用过 CUDA 设备序号（ordinal）和主机枚举 index。进程按 `CUDA_VISIBLE_DEVICES` 的次序枚举可见 GPU，列表第一项在它眼里就是 `cuda:0`。本例每台主机的四个 Worker 共享同一份可见列表，应用才可以有规律地让每个 Worker 用 `LOCAL_RANK` 选设备。设备的 UUID（通用唯一标识符）用于对应同一张卡，PCI BDF（域、总线、设备、功能）用于定位它在该主机上的 PCI 接入位置。 [CUDA 13.2.1 · CUDA_VISIBLE_DEVICES](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html#cuda-visible-devices) · [NVIDIA SMI · 设备标识](https://docs.nvidia.com/deploy/nvidia-smi/)

下表是**教学记录**。A 上的可见列表用 UUID 按 `A0,A1,A2,A3` 排列，B 上按 `B2,B0,B3,B1` 排列。`A0` 等均代指完整 UUID。主机侧用 `nvidia-smi --query-gpu=index,uuid,pci.bus_id --format=csv` 查 index 与 BDF，进程侧在选卡后记录实选 CUDA 序号和设备身份，再按主机及 UUID 合表。所有行均属 `run_id=ai05-demo-001`、`attempt=0`，两项分别取自 `TORCHELASTIC_RUN_ID`、`TORCHELASTIC_RESTART_COUNT`。

| 主机 | PID | 全局 Rank | Local Rank | 实选 CUDA 序号 | 主机 index | UUID 简称 | PCI BDF |
| --- | ---: | ---: | ---: | ---: | ---: | --- | --- |
| A | 2100 | 0 | 0 | 0 | 0 | A0 | `0000:21:00.0` |
| A | 2101 | 1 | 1 | 1 | 1 | A1 | `0000:41:00.0` |
| A | 2102 | 2 | 2 | 2 | 2 | A2 | `0000:81:00.0` |
| A | 2103 | 3 | 3 | 3 | 3 | A3 | `0000:c1:00.0` |
| B | 3200 | 4 | 0 | 0 | 2 | B2 | `0000:81:00.0` |
| B | 3201 | 5 | 1 | 1 | 0 | B0 | `0000:21:00.0` |
| B | 3202 | 6 | 2 | 2 | 3 | B3 | `0000:c1:00.0` |
| B | 3203 | 7 | 3 | 3 | 1 | B1 | `0000:41:00.0` |

Rank 4 在 B，读取 `LOCAL_RANK=0`，选择 `cuda:0`，于是命中 B 的列表第一项 B2，即 B 主机 index 2。A 的 A2 与 B 的 B2 恰好有同一个 BDF，查拓扑时要带主机名。如果平台给每个 Worker 只暴露它自己的卡，每个进程的可见列表都只有一个元素，它们可以同时使用 `cuda:0`。这种设置应逐进程核对实际 UUID。 [CUDA 13.2.1 · CUDA_VISIBLE_DEVICES](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html#cuda-visible-devices)

反过来，若在共享可见列表的本例中，B 上 Rank 4 和 5 都硬编码选择 `cuda:0`，两者会同时选到 B2，B0 空置。NCCL（NVIDIA Collective Communications Library）是 PyTorch 可选用的 GPU 间通信库。PyTorch 用 NCCL 在多进程间交换 GPU 数据时，要求每个进程独占它使用的 GPU，共用设备可能导致无效用法或死锁。核实独占时，要在同一轮的记录中检查实选 UUID 是否重复。 [NCCL 2.31.2 · 概览](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/overview.html) · [PyTorch 2.9 · init_process_group](https://docs.pytorch.org/docs/2.9/distributed.html#torch.distributed.init_process_group)

## 同一 Worker 在子组里的编号

PyTorch 用**进程组**定义一次通信有哪些进程参加以及各自的组内 Rank。默认进程组包括全部八个 Worker。应用也可以取其中几个成员创建子组，例如 `[1,5]` 跨 A、B 两台机器，全局 Rank 1、5 分别是组内 Rank 0、1。换成子组 `[0,2,5]`，全局 Rank 5 在组内就是 2，在 B 节点的 Local Rank 仍是 1。组内 Rank 决定谁提供广播数据、谁得到某一片结果。实际调用可用 `get_group_rank` 核对转换。 [PyTorch 2.9 · 进程组与 Rank 映射](https://docs.pytorch.org/docs/2.9/distributed.html#groups)
