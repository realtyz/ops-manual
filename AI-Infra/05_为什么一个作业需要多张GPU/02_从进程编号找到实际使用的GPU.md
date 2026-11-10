# 从进程编号找到实际使用的 GPU

两台机器各有一个进程报告 `LOCAL_RANK=0`。在两节点各四卡的作业中，这通常是两个不同的 Worker，各自在本机选卡。要核对用卡，先确定编号的范围，再把进程看到的设备列表和主机侧设备身份对应起来。

## 同一个进程有哪些编号

这里用 PyTorch 2.9 的 `torchrun`。每个 Linux 节点启动一个 Agent，由它启动并监控本机的四个训练进程。每个进程使用一张 GPU。Agent 给 Worker 提供的几个环境变量各有自己的范围。 [PyTorch 2.9 · torchrun](https://docs.pytorch.org/docs/2.9/elastic/run.html#environment-variables)

| 编号或数量 | 范围 | 本轮示例 |
| --- | --- | --- |
| `RANK` | 本轮整个 Worker 组 | A 上为 0—3，B 上为 4—7 |
| `WORLD_SIZE` | 本轮整个 Worker 组的成员数 | 8 |
| `LOCAL_RANK` | 本节点的本地 Worker 组 | 两台各有 0—3 |
| `LOCAL_WORLD_SIZE` | 每个本地 Worker 组的成员数 | 4 |
| 组内 Rank | 某个进程组内的位置 | 子组 `[1,5]` 中，全局 1 对应组内 0，全局 5 对应组内 1 |

子组 `[1,5]` 横跨两台机器。全局 Rank 5 的成员同时拥有本地 Rank 1 和该子组的组内 Rank 1，两者在此例恰好相等。换成子组 `[0,2,5]`，它就成了组内 Rank 2，本地 Rank 仍是 1。调用集合操作时要先找通信组，再查该 API 的 Rank 参数使用哪种编号。 [PyTorch 2.9 · get_group_rank](https://docs.pytorch.org/docs/2.9/distributed.html#torch.distributed.get_group_rank)

## 从可见设备列表追到物理卡

在同一节点上，本例的四个 Worker 继承相同的 `CUDA_VISIBLE_DEVICES`。应用读取 `LOCAL_RANK` 后显式选择同序号的 CUDA 设备。CUDA 会按可见列表重新编号。列表中的第一张卡在进程里就是 `cuda:0`，因此调换列表次序会改变 `cuda:0` 对应的 GPU。用 UUID 指定列表可以固定想要的设备和顺序，再用 BDF（PCI 域、总线、设备、功能）对应到当前主机上的位置。 [CUDA 13.2.1 · CUDA_VISIBLE_DEVICES](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html#cuda-visible-devices)

以下是**自拟设备记录**，模拟同一轮 Linux 两节点、每节点四张 NVIDIA 整卡的环境。UUID 栏用简称 `A0…A3`、`B0…B3` 表示八个不同的完整 GPU UUID。B 节点设想通过 `nvidia-smi --query-gpu=index,uuid,pci.bus_id --format=csv` 得到如下设备清单，表中统一了 BDF 的域号宽度。

| B 主机 index | UUID 简称 | PCI BDF |
| ---: | --- | --- |
| 0 | B0 | `0000:21:00.0` |
| 1 | B1 | `0000:41:00.0` |
| 2 | B2 | `0000:81:00.0` |
| 3 | B3 | `0000:c1:00.0` |

A 节点将 `CUDA_VISIBLE_DEVICES` 按 `A0,A1,A2,A3` 的顺序设为四个完整 UUID。B 节点按 `B2,B0,B3,B1` 的顺序设置。下表把启动器交给进程的编号、应用记录的选卡和主机清单合在一起。每行共有 `run_id=ai05-demo-001`、`attempt=0`，其中 `attempt` 对应 `TORCHELASTIC_RESTART_COUNT`。

| 主机 | PID | 全局 Rank | Local Rank | 实选 CUDA ordinal | 主机 index | UUID 简称 | PCI BDF |
| --- | ---: | ---: | ---: | ---: | ---: | --- | --- |
| A | 2100 | 0 | 0 | 0 | 0 | A0 | `0000:21:00.0` |
| A | 2101 | 1 | 1 | 1 | 1 | A1 | `0000:41:00.0` |
| A | 2102 | 2 | 2 | 2 | 2 | A2 | `0000:81:00.0` |
| A | 2103 | 3 | 3 | 3 | 3 | A3 | `0000:c1:00.0` |
| B | 3200 | 4 | 0 | 0 | 2 | B2 | `0000:81:00.0` |
| B | 3201 | 5 | 1 | 1 | 0 | B0 | `0000:21:00.0` |
| B | 3202 | 6 | 2 | 2 | 3 | B3 | `0000:c1:00.0` |
| B | 3203 | 7 | 3 | 3 | 1 | B1 | `0000:41:00.0` |

例如，Rank 4 的选卡过程是 `LOCAL_RANK=0 → cuda:0 → 可见列表第一项 B2 → B 主机 index 2`。两台机器可以有相同的 BDF，所以核对时应带上主机名。实际采集时应保留完整 GPU UUID。若不同 Worker 的可见列表不同，要逐进程读取列表和实选设备，不能把上表的同序号选卡规则直接套上去。 [CUDA 13.2.1 · Environment Variables](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html#cuda-visible-devices) · [NVIDIA SMI · Device Enumeration](https://docs.nvidia.com/deploy/nvidia-smi/)

## 用实选结果检查独占

假设 B 的两个 Worker 都把设备硬编码为 `cuda:0`。它们仍有不同的 `RANK` 和 `LOCAL_RANK`，但实选 UUID 都是 B2，原本分给 Rank 5 的 B0 空着。在 PyTorch 的多进程 NCCL 场景里，每个进程须独占自己使用的 GPU，跨进程共享同一 GPU 可能导致 NCCL 无效使用或死锁。进程数等于卡数，只能说明数量对得上。要核实独占，按同一轮尝试逐个检查选卡后的 UUID。 [PyTorch 2.9 · init_process_group](https://docs.pytorch.org/docs/2.9/distributed.html#torch.distributed.init_process_group)

重启后应重新采集这一映射。`RANK` 可在重新组队时变化，PID 也会重新分配。排查历史记录时用运行 ID、尝试编号和主机把记录分开，再把当轮 Rank 对应到当时的 PID 和 GPU。 [PyTorch 2.9 · torchrun Important Notices](https://docs.pytorch.org/docs/2.9/elastic/run.html#important-notices)
