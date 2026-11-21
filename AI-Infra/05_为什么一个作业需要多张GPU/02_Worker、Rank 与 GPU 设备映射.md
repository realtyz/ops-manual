# Worker、Rank 与 GPU 设备映射

当一个分布式作业从单机四卡扩展到双机八卡后，日志和报错里瞬间会冒出一堆容易打架的编号：每个节点上都有 `cuda:0`，每个节点也都有 `LOCAL_RANK=0`，而全局编号又从 `0` 排到 `7`。一旦进程编号与物理显卡没有对齐，轻则查错拓扑分支，重则两个进程撞到同一张卡上直接把集群跑死锁。

要把多机通信落回物理硬件，必须先彻底理顺三层身份：**节点上的进程角色、作业内的逻辑编号，以及物理 GPU 的唯一标识**。

## Worker、Agent 与四类核心编号

以 PyTorch 的官方分布式启动器 `torchrun` 为例，在双节点八卡集群（`node-1`、`node-2` 各 4 张整卡）上运行作业时，每个节点会有两类进程协同工作：

- **Agent**：由 `torchrun` 在每台节点上启动的本地守护进程，负责与其他节点的 Agent 完成会合、拉起并监控本机训练进程。
- **Worker**：真正执行模型计算与通信的 Linux 进程。主流配置是一卡一进程，即每节点 4 个 Worker，每个 Worker 独占一张可见 GPU。

为了让分布在不同机器上的 Worker 知道“我是谁、团队有多大、本机有几人”，**Rank** 被定义为成员在某个通信组内从 `0` 开始的连续编号。`torchrun` 会合后会向每个 Worker 注入四个核心环境变量：

| 环境变量 | 定义与作用域 | 双节点八卡示例 |
| --- | --- | --- |
| `RANK` | **全局 Rank**：当前 Worker 在本轮整个作业组内的编号 | `node-1` 分得 `0–3`，`node-2` 分得 `4–7` |
| `WORLD_SIZE` | **全局规模**：本轮整个作业组的 Worker 总数 | `8` |
| `LOCAL_RANK` | **本地 Rank**：当前 Worker 在所在节点内的编号 | `node-1` 与 `node-2` 各自为 `0–3` |
| `LOCAL_WORLD_SIZE` | **本地规模**：当前节点内启动的 Worker 数量 | 每节点各 `4` 个 |

需要注意：`node-1` 的全局 `Rank 0` 与 `node-2` 的全局 `Rank 4`，它们的 `LOCAL_RANK` 都是 `0`。要在系统层唯一锁定进程，必须同时记下**主机名与 PID**。

由于弹性重启会产生新 PID 甚至重分 Rank，采集排障记录时还应附带本轮运行 ID（`TORCHELASTIC_RUN_ID`）与重启计数（`TORCHELASTIC_RESTART_COUNT`）。

## 从逻辑序号落回物理 GPU：防重排与防撞卡

如 [第 04 章](../04_单机多卡与GPU数据路径/00_内容大纲.md) 所述，进程眼里的 `cuda:0` 由 `CUDA_VISIBLE_DEVICES` 顺序决定，而非宿主机静态序号。只有同节点 Worker 共享一致的可见列表，按 `LOCAL_RANK` 选卡才互不冲突。

看下面这份双节点八卡（`run_id=train-demo-001`、`attempt=0`）的实际映射表。`node-1` 的可见列表按正常顺序排列（`GPU-a001` 至 `GPU-a004`），而 `node-2` 的 `CUDA_VISIBLE_DEVICES` 按重排顺序暴露为 `GPU-b003,GPU-b001,GPU-b004,GPU-b002`：

| 节点 | PID | 全局 `RANK` | `LOCAL_RANK` | 实选 CUDA 序号 | 宿主机 index | 物理 UUID（简称） | PCI BDF |
| --- | ---: | ---: | ---: | ---: | ---: | --- | --- |
| `node-1` | `2100` | `0` | `0` | `0` | `0` | `GPU-a001` | `0000:21:00.0` |
| `node-1` | `2101` | `1` | `1` | `1` | `1` | `GPU-a002` | `0000:41:00.0` |
| `node-1` | `2102` | `2` | `2` | `2` | `2` | `GPU-a003` | `0000:81:00.0` |
| `node-1` | `2103` | `3` | `3` | `3` | `3` | `GPU-a004` | `0000:c1:00.0` |
| `node-2` | `3200` | `4` | `0` | `0` | `2` | `GPU-b003` | `0000:81:00.0` |
| `node-2` | `3201` | `5` | `1` | `1` | `0` | `GPU-b001` | `0000:21:00.0` |
| `node-2` | `3202` | `6` | `2` | `2` | `3` | `GPU-b004` | `0000:c1:00.0` |
| `node-2` | `3203` | `7` | `3` | `3` | `1` | `GPU-b002` | `0000:41:00.0` |

读这张表时，有三个线上排障必须盯紧的细节：

1. **序号漂移靠 UUID 锚定**：`node-2` 上全局 `Rank 4` 读取 `LOCAL_RANK=0` 选中 `cuda:0`，落到的是重排列表第一项 `GPU-b003`（宿主机 index `2`），而非 index `0`。
2. **BDF 必须绑定主机名解读**：`node-1` 的 `GPU-a003` 与 `node-2` 的 `GPU-b003` 在各自服务器上恰好拥有相同的 `0000:81:00.0`，跨机排查拓扑绝不能只抄 BDF 漏掉主机名。
3. **单卡隔离容器的特殊情形**：如果容器编排平台（如 Kubernetes）只给每个 Worker 暴露分配给它的那 1 张卡，那么所有进程眼里都只有 `cuda:0`，此时更要逐进程核对实选 UUID 是否两两不同。

线上启动脚本中，通常在选卡后立刻打印一行包含主机名、PID、编号与真实 UUID 的自检日志，方便第一时间捕获撞卡故障：

```bash
# 宿主机侧：采集本机 index、完整 UUID 与 PCI BDF 对照基准
$ nvidia-smi --query-gpu=index,uuid,pci.bus_id --format=csv,noheader
0, GPU-b001-..., 00000000:21:00.0
1, GPU-b002-..., 00000000:41:00.0
2, GPU-b003-..., 00000000:81:00.0
3, GPU-b004-..., 00000000:C1:00.0
```

```python
# Worker 进程侧：选卡后立即落盘 (run_id, attempt, host, pid, rank, local_rank, uuid)
import os, socket, torch

local_rank = int(os.environ["LOCAL_RANK"])
torch.cuda.set_device(local_rank)
props = torch.cuda.get_device_properties(local_rank)
print(
    f"run={os.environ.get('TORCHELASTIC_RUN_ID')} attempt={os.environ.get('TORCHELASTIC_RESTART_COUNT')} "
    f"host={socket.gethostname()} pid={os.getpid()} rank={os.environ['RANK']} "
    f"local_rank={local_rank} cuda={local_rank} uuid={props.uuid}",
    flush=True,
)
```

> [!warning] 常见误区：两个 Worker 误选同一张 GPU 会直接触发死锁或非法用法
> 在共享可见设备列表的节点上，若应用误将所有 Worker 都硬编码成 `cuda:0`，`node-2` 上的全局 `Rank 4` 与 `Rank 5` 就会同时占住 `GPU-b003`，而 `GPU-b001` 彻底空置。
>
> PyTorch 使用 NCCL（详见 [第 08 章](../08_NCCL如何组织GPU间的通信/00_内容大纲.md)）进行多进程通信时，严格要求**每个进程独占所绑定的 GPU**——同卡冲突极易导致非法调用或集合通信挂死；核实独占的铁证，是检查全组所有 Worker 实选 UUID 毫无重复。

## 全局 Rank 与子通信组的“组内 Rank”

多卡作业很少只把全部成员塞进一个大群里通信。随着模型越来越复杂，同一批 Worker 往往要同时参与多个维度的数据交换（例如一组负责同机张量并行、另一组负责跨机梯度同步）。

PyTorch 用**进程组（Process Group）** 定义“哪些进程一起参与通信”以及它们在这个小群里的编号，称为**组内 Rank（Group Rank）**：

- **默认进程组**：包含全部 8 个成员，此时组内 Rank 等于全局 `RANK`（`0–7`）。
- **两成员跨机子组 `[1, 5]`**：全局 `Rank 1`（位于 `node-1`）在该子组内是 `组内 Rank 0`，全局 `Rank 5`（位于 `node-2`）在该子组内是 `组内 Rank 1`。
- **三成员子组 `[0, 2, 5]`**：同一个全局 `Rank 5`，在这个新子组里变成了 `组内 Rank 2`，而它在 `node-2` 上的 `LOCAL_RANK` 依然是 `1`。

组内 Rank 直接决定了集合通信中“谁当广播根、谁负责接收第几片分块”。编写或排查子组通信代码时，可随时通过 `dist.get_group_rank(group, global_rank)` 核对全局编号到组内编号的精确换算。

> [!summary] 核心结论
> 记住四组身份的边界：**主机名 + PID** 定位 Linux 进程，**UUID + 完整 BDF** 定位物理显卡，**`LOCAL_RANK`** 解决节点内选卡不撞车，**全局 `RANK` 与组内 Rank** 定义分布式通信拓扑。有了清晰的成员编号，下一篇我们就可以把目光聚焦到通信本身——拆解五大核心集合通信原语究竟在搬运什么数据。

> [!info]- 参考资料
> - **PyTorch `torchrun` 启动、环境变量与进程组**：[PyTorch 2.9 · `torchrun` (Elastic Launch)](https://docs.pytorch.org/docs/2.9/elastic/run.html)、[Environment Variables](https://docs.pytorch.org/docs/2.9/elastic/run.html#environment-variables)、[Important Notices](https://docs.pytorch.org/docs/2.9/elastic/run.html#important-notices)、[`init_process_group`](https://docs.pytorch.org/docs/2.9/distributed.html#torch.distributed.init_process_group)、[Process Groups & Rank Mapping](https://docs.pytorch.org/docs/2.9/distributed.html#groups)
> - **CUDA 设备可见性与物理标识**：[CUDA 13.2.1 · `CUDA_VISIBLE_DEVICES`](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html#cuda-visible-devices)、[NVIDIA SMI · Device Identifiers](https://docs.nvidia.com/deploy/nvidia-smi/)
> - **NCCL 多进程独占设备约定**：[NCCL 2.27.3 · Overview](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/overview.html)
