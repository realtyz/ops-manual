# MIG 硬件实例、Profile 与放置

当在线大模型推理承诺严格的 `TTFT P99` 与 `TPOT P99` 服务等级目标时，Time-Slicing 和 CUDA MPS 都绕不开同一个硬伤：无论软件怎么调配时间片或线程比例，同卡邻居依然挤在同一套 L2 缓存、显存控制器和 HBM 带宽总线上。

要彻底掐断邻居的显存溢出连坐和 HBM 带宽争抢，必须在硬件层把一张物理 GPU 划成几块彼此独立的“微型 GPU”。这就是 **MIG（Multi-Instance GPU，多实例 GPU）**。它在支持的 NVIDIA 数据中心卡（如 A100、H100）上由驱动执行物理切片，再把切出的实例像独立显卡一样交给容器。

## GI 与 CI：硬件隔离的两道核心边界

MIG 用**切片（slice）** 度量可切分的硬件配额，并以两层对象组织硬件隔离。以单张 A100-SXM4-40GB 切出两个 `3g.20gb` 实例为例，每个交付给容器的 MIG 设备由一个 **GI** 和其内部的一个 **CI** 共同构成：

1. **GPU Instance（GI，GPU 实例）**：真正的硬件硬隔离边界。每个 GI 独占一组计算切片、显存切片与对应引擎，同时**物理切分 L2 缓存分区、显存控制器和 HBM 内存带宽通道**。分属不同 GI 的推理副本彻底绝缘，谁也抢不到对方的容量和带宽。
2. **Compute Instance（CI，计算实例）**：位于父 GI 内部的计算子分区。默认一个 GI 只建一个占满全量计算切片的 CI；如果强行在同一个 GI 里切出多个 CI，它们各自拿着独立计算切片，**却共用父 GI 的同一份显存容量与内存总线**。

下面这幅对照图把两者的差异画得一清二楚。左侧是推荐的在线推理隔离布局，右侧则是极易踩坑的“同 GI 多 CI”误区：

![左图任务 A 与任务 B 分别使用 GPU 0 上 GI 0 与 GI 1 中的 CI 0，各自访问所属 GI 的独立显存。右图任务 A 与任务 B 同处 GI 0 下的 CI 0 与 CI 1，共用 GI 0 显存，而 GI 1 空闲。](images/fig-14-03-01-mig-gi-and-ci.png)

*图 14-03-01：同一张物理卡上的跨 GI 隔离与同 GI 多 CI 共享布局对照。实线表示任务与 CI 的绑定，虚线表示对所属 GI 显存的访问路径。*

对照 `图 14-03-01` 可以一眼看出关键结论：如果两套在线推理服务需要独立的 KV Cache 容量边界和稳定的 Decode HBM 带宽，就必须像左图那样落进 `GI 0` 和 `GI 1` 这两个**不同 GI**；如果像右图那样塞进同一个 `GI 0` 下的 `CI 0` 与 `CI 1`，两份 KV Cache 依然在同一池显存里互相挤压，HBM 带宽也会照常打架。

> [!warning] 常见误区：跨 GI 隔离了硬件资源，却没有拆开物理卡故障域
> 不同 GI 之间可以挡住大部分局部计算故障。例如在 A100 故障处置规范中，`Xid 94` 属于可收束在受影响实例内的受限错误；但一旦出现 `Xid 95` 这类波及全局硬件的严重故障，仍必须排空同卡其他 GI 上的在线副本并复位整张物理卡。更何况整机供电与宿主机宕机同样会一锅端。

## A100 Profile 规格、命名与切片核算

**Profile（规格档位）** 是硬件预先固化好的“显存切片 + 计算切片 + 数据搬运引擎”组合，不能像虚拟机那样任意填写数字。下表列出 A100-SXM4-40GB 的主流 GI Profile（整卡共 **8 个内存切片**与 **7 个计算切片**，Copy Engine 负责异步数据搬运）：

| GI Profile | Profile ID | 内存切片占比 | SM 切片占比 | Copy Engine 数 | 单卡最大同规格实例数 |
| --- | ---: | ---: | ---: | ---: | ---: |
| `1g.5gb` | `19` | `1/8` | `1/7` | `1` | `7` |
| `2g.10gb` | `14` | `2/8` | `2/7` | `2` | `3` |
| `3g.20gb` | `9` | `4/8` | `3/7` | `3` | `2` |
| `4g.20gb` | `5` | `4/8` | `4/7` | `4` | `1` |
| `7g.40gb` | `0` | `8/8` | `7/7` | `7` | `1` |

仔细看 `3g.20gb` 与 `4g.20gb` 的切片占比，马上能算出一笔很有意思的卡内利用率账：

- **两个 `3g.20gb` 对称切分**：

$$M_{\text{used}} = 2 \times \frac{4}{8} = \frac{8}{8}\ \text{（内存切片用满）}, \qquad C_{\text{used}} = 2 \times \frac{3}{7} = \frac{6}{7}\ \text{（剩余 1 个 SM 切片）}$$

  虽然卡上还剩 `1/7` 计算切片，但内存切片已经归零，连最小的 `1g.5gb` 都放不下——这 `1/7` 算力就此成为不可回收的**固定切片损耗**。

- **一个 `4g.20gb` + 一个 `3g.20gb` 混搭**：

$$M_{\text{used}} = \frac{4}{8} + \frac{4}{8} = \frac{8}{8}, \qquad C_{\text{used}} = \frac{4}{7} + \frac{3}{7} = \frac{7}{7}\ \text{（算力与显存双满）}$$

  同等显存切片下吃满了全部 7 个计算切片，把较高算力的 `4g.20gb` 留给高并发 Prefill 副本，正是生产推理池常用的非对称切法。

在命名规则上，当 `3g.20gb` 的 GI 内只建一个占满全量计算切片的 CI 时，设备名直接简写为 `3g.20gb`；如果只把其中 1 个计算切片单独划成一个 CI，就会出现 `1c.3g.20gb` 这种完整名称，后面的 `3g.20gb` 仍指代它所依托的父 GI。

> [!note] 单位与卡型换算提醒
> Profile 名字里的 `20gb`、`40gb` 只是厂商的标称规格拼写。评估模型权重和 KV Cache 能不能装下，必须以运行时上报的实际可用 MiB/GiB 容量为准。此外，不同卡型的档位与合法位置差异很大，例如 A100 80GB 和 H100 80GB 对应的半卡大规格为 `3g.40gb` 与 `4g.40gb`，上线前务必在目标卡型上实查规格表。

## 合法放置窗口与卡内碎片

凑够剩余切片数，依然可能无法创建目标实例。受芯片内部显存控制器与交叉开关的物理布线限制，每种 MIG Profile 只能落在固定的连续**放置窗口（Placement）** 内。

在 A100-SXM4-40GB 上，`3g.20gb` 的合法窗口标记为 `{0,4}:4`——意思是从位置 `0` 起跨 4 格（占用 `0–3`），或从位置 `4` 起跨 4 格（占用 `4–7`）。普通 `1g.5gb` 的合法起点是 `0–6`（跨度为 `1`），而位置 `7` 只能作为大规格窗口的尾部被一并占走。

下面两个状态都已经在卡上建了两个 `1g.5gb`，剩余切片数完全一致，结局却截然不同：

| 已有两个 `1g.5gb` 的起点 | 窗口 `0–3` 状态    | 窗口 `4–7` 状态    | 能否再建 `3g.20gb`      |
| ----------------- | -------------- | -------------- | ------------------- |
| 起点 `0` 与起点 `4`    | 位置 `0` 被占，窗口截断 | 位置 `4` 被占，窗口截断 | **不能**（两个合法窗口均重叠）   |
| 起点 `0` 与起点 `1`    | 位置 `0、1` 被占    | 完整空闲           | **可以**（落入 `4–7` 窗口） |

两种布局都剩下 6 个位置和 5 个计算切片，前一种却因为两颗小实例刚好钉在两个大窗口的门坎上，彻底堵死了大规格推理副本的落脚点——这就是 MIG 特有的**卡内放置碎片**。

现场运维核对规格、创建实例与排查碎片时，按“查规格 ID → 查合法窗口 → 按从大到小顺序创建 → 查实际占位”四步执行即可：

```sh
# 1. 列出支持的 GI Profile 名称、Profile ID 与剩余可建数
$ nvidia-smi mig -lgip
# GPU  Name             ID  Instances   Memory     P2P    SM    DEC   ENC
#  0   MIG 1g.5gb       19     7/7      4.75 GiB   No     14     0     0
#  0   MIG 2g.10gb      14     3/3      9.75 GiB   No     28     1     0
#  0   MIG 3g.20gb       9     2/2     19.62 GiB   No     42     2     0
#  0   MIG 4g.20gb       5     1/1     19.62 GiB   No     56     2     0

# 2. 查看各 Profile 的合法起始位置与跨度（Placement {Start}:Size）
$ nvidia-smi mig -lgipp
# GPU 0 Profile ID 19 Placements: {0,1,2,3,4,5,6}:1
# GPU 0 Profile ID  9 Placements: {0,4}:4    # 3g.20gb 仅允许落入 0-3 或 4-7

# 3. 创建两个 3g.20gb GI 并自动配齐默认 CI（-C）；混建多规格时须先大后小，避免窗位冲突
$ sudo nvidia-smi mig -cgi 9,9 -C
Successfully created GPU instance ID 1 on GPU 0 using profile MIG 3g.20gb (ID 9)
Successfully created GPU instance ID 2 on GPU 0 using profile MIG 3g.20gb (ID 9)

# 4. 查看已生效 GI 的实际占位（Start:Size）
$ nvidia-smi mig -lgi
+----------------------------------------------------+
| GPU instances:                                     |
| GPU   Name          Profile  Instance   Placement  |
|                       ID        ID       Start:Size|
|====================================================|
|   0  MIG 3g.20gb       9        1          0:4     |  # 占据窗口 0-3
|   0  MIG 3g.20gb       9        2          4:4     |  # 占据窗口 4-7
+----------------------------------------------------+
```

> [!tip] 工程直觉：混建 MIG 时遵循“先大后小”原则
> 同一张卡上混建多种规格（例如一个 `3g.20gb`、一个 `2g.10gb` 和两个 `1g.5gb`）时，如果把小规格写在 `-cgi` 参数前面，小实例可能先占掉跨窗口的关键位置导致后面的大规格报 `Insufficient Resources`。把大 Profile ID 排在前面（如 `-cgi 9,14,19,19`）能大幅降低布局冲突。

## 五维工程评估：MIG 在在线推理池中的定位

把 MIG 放进面向在线推理的五维工程标尺，它的优势和代价都极其鲜明：

| 工程维度 | MIG（硬件级物理实例切分） |
| --- | --- |
| **显存硬隔离** | **极强（物理级）**。各 GI 独占显存切片与内存控制器，邻居 KV Cache 突发绝不会波及本实例。 |
| **算力与 HBM 带宽干扰** | **极低（物理级）**。SM 切片、L2 缓存与 HBM 通道按 GI 硬件切开，邻居 Prefill 洪峰与 Decode 满载带宽均不穿透。 |
| **P99 尾延迟表现** | **极佳且可预测**。同卡并跑的 TTFT P99 与 TPOT P99 几乎与单跑一致，是高 SLO 生产推理池的黄金基线。 |
| **卡内碎片率** | **中高**。只能按离散 Profile 档位切分（例如 12 GiB 模型放不进 `2g.10gb`，放进 `3g.20gb` 会空置近 8 GiB），且存在合法窗口放置碎片。 |
| **运维灵活性** | **较低（静态预建模式）**。修改节点布局需排空目标卡上的活跃推理副本，重新切分后再回流流量。 |

> [!summary] 核心结论
> MIG 用一部分显存颗粒度浪费和重配置灵活性，换来了同卡多租户在线推理最稀缺的资产——**物理级显存容量、HBM 带宽与 P99 尾延迟的确定性**。接下来看 Kubernetes 如何把整卡、Time-Slicing 共享名额和预建 MIG 实例统一编排成节点资源库存。

> [!info]- 参考资料
> - **MIG 原理、支持型号与硬件隔离**：[NVIDIA MIG 620 简介与内存系统](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/introduction.html)、[NVIDIA MIG 支持的 GPU 与系统](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/supported-gpus.html)、[NVIDIA MIG 620 实例划分与术语](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/concepts.html#partitioning)
> - **Profile 规格、命名与合法放置**：[NVIDIA MIG 620 A100 Profile 表](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/supported-mig-profiles.html#a100-mig-profiles)、[NVIDIA MIG 620 全卡型 Profile 汇总](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/supported-mig-profiles.html)、[NVIDIA MIG 610 设备命名规范](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/610/mig-device-names.html)、[NVIDIA MIG 620 Profile 放置原则](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/concepts.html#profile-placement)
> - **命令行枚举与创建示例**：[NVIDIA MIG 620 实例几何与位置列表](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/getting-started-with-mig.html#instance-geometry)、[NVIDIA MIG 620 创建实例样例](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/getting-started-with-mig.html)
> - **MIG 硬件故障收束范围**：[NVIDIA GPU 故障处置指南](https://docs.nvidia.com/deploy/gpu-debug-guidelines/620/gpu-node-triage.html)
