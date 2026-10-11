---
tags:
  - AI-Infra
  - NVLink
  - MNNVL
  - IMEX
  - 原理
created: 2026-10-11
---

# 多机 NVLink 与 IMEX 域

多机 NVLink（Multi-Node NVLink，MNNVL）把原本止步于机箱内的 NVLink 域扩展到多台机器。它带来的直接变化是：两个不在同一操作系统实例里的 GPU，可以在满足条件时被通信库视为可跨节点访问的 GPU，而不是只能通过 RDMA 或套接字转发。Internode Memory Exchange（IMEX）则是这套跨节点内存交换所需的管理与诊断入口：它维护跨节点可导入导出的内存句柄通道，使一个节点上的 CUDA fabric 句柄能被另一个节点使用。

**硬件上存在 MNNVL 不等于通信一定走 MNNVL。**NCCL 要使用它，至少要同时满足以下条件：

| 前提 | 含义 | 不成立时的结果 |
| --- | --- | --- |
| CUMEM 支持 | NCCL 使用 CUDA `cuMem*` 分配与句柄能力 | MNNVL 不启用，或显式要求时初始化失败 |
| 所有 rank 支持 fabric handle | 每个参与进程的 GPU 都能导出 / 导入 fabric 句柄 | 任一 rank 不支持，整组不能按 MNNVL 路径工作 |
| fabric 状态完成 | 每张 GPU 的 fabric 信息都达到 completed 状态 | NCCL 判定 MNNVL 不可用 |
| 相同 fabric UUID | 多节点属于同一个 NVLink 域 | 不会被视为同一 MNNVL 域 |
| 相同 clique，或明确允许跨 clique | 同一 UUID 内再按 clique 分组；默认只把相同 `<UUID, cliqueId>` 视为同一 NVLink 域 | 相同 UUID 但不同 clique 的 rank 不按同域处理 |
| IMEX 通道可用 | FABRIC 句柄能导出并导入，典型检查对象包括 IMEX 通道配置 | MNNVL 能力存在但不可用，NCCL 给出 IMEX 配置告警 |

NCCL 侧的开关是 `NCCL_MNNVL_ENABLE`：

| 取值 | 语义 |
| --- | --- |
| `0` | 禁用 MNNVL 支持 |
| `1` | 启用；若系统、驱动或资源不支持，初始化失败 |
| `2` | 自动探测；不支持或资源不可分配时不失败，退回其他路径 |

`NCCL_MNNVL_UUID` 与 `NCCL_MNNVL_CLIQUE_ID` 可以覆盖域标识：UUID 通常由 Fabric Manager 分配，clique 也可以由 Fabric Manager 分配；人工覆盖通常只用于软分区实验或特定部署。把它们当成「调优参数」随手改，会把真实的域拓扑和通信库看到的域拓扑拆开。

## 域、UUID 与 clique 的分层

MNNVL 里有两个容易混淆的标识：

| 标识 | 回答什么 | 由谁维护 | 不一致时的现象 |
| --- | --- | --- | --- |
| fabric UUID / cluster UUID | 这些 GPU 是否属于同一个跨节点 NVLink 域 | Fabric Manager 侧的域信息 | 不同 UUID 的 rank 不会被当成同一 MNNVL 域 |
| clique ID | 同一 UUID 内部的连通分组或软分区 | Fabric Manager 分配，NCCL 环境变量可覆盖 | 相同 UUID 但 clique 不同时，默认不按同 clique 处理 |

把一次跨节点通信的判定走完：

| 步骤 | NCCL 读到的对象 | 判定 |
| --- | --- | --- |
| 1 | 每张 GPU 的 fabric 信息 | 状态应为 completed；healthMask 不应报告降级带宽 |
| 2 | 各 rank 的 fabric UUID | UUID 相同才属于同一跨节点 NVLink 域 |
| 3 | 各 rank 的 cliqueId | 相同 clique 才默认视为同组；不同 clique 需要 cross-clique 能力 |
| 4 | FABRIC 句柄导出 / 导入 | IMEX 通道可用，句柄能被对端导入 |
| 5 | 拓扑图中的 peer 关系 | 满足以上条件后，跨节点 GPU 可按 MNNVL / CUDA P2P 路径处理 |

NCCL 日志中能看到的典型字段包括：

```text
MNNVL busId ... fabric UUID ... cliqueId ... state ... healthMask ... degradedBw ...
MNNVL 1 cliqueId ... cliqueSize ... cliqueRank ... nvlDomainSize ...
```

这些字段的含义分别是：

| 字段 | 是什么 | 判据 |
| --- | --- | --- |
| `busId` | 这条日志对应的 GPU PCI 总线位置 | 用于把日志行映射回具体 GPU |
| `fabric UUID` | 跨节点 NVLink 域标识 | 参与同一 MNNVL 通信的 rank 应一致 |
| `cliqueId` | UUID 内的分组标识 | 默认同 clique 才被视为同组 |
| `state` | GPU fabric 信息的状态 | 应为 completed |
| `healthMask` | NVLink fabric 健康位图 | 出现 degraded bandwidth 等位时不能当作健康域 |
| `degradedBw` | 降级带宽的描述 | 非空或报告降级时，性能结论不能按健康域解释 |
| `cliqueSize` | 当前通信中同 clique 的 rank 数 | 应与预期的 MNNVL 分组一致 |
| `nvlDomainSize` | 同一 UUID 域内的 rank 数 | 大于 cliqueSize 时说明存在同域但不同 clique 的 rank |

当 MNNVL 能力存在但 IMEX 通道不可用时，NCCL 的告警会指向两类对象：`/dev/nvidia-caps-imex-channels` 与 `nvidia-imex-ctl -N` 一类的 IMEX 配置检查入口。这条告警的价值不是给出最终根因，而是把诊断对象从「网卡」转移到「跨节点内存句柄通道」。

## MNNVL、RDMA 与 NVLS 的分工

三者都能影响跨机集合通信，但作用面不同：

| 路径或机制 | 它提供什么 | 与 MNNVL 的关系 |
| --- | --- | --- |
| RDMA / IB / RoCE | 通过网卡与 fabric 传输跨机数据 | MNNVL 不可用或不适合时，通信库可能退回网络路径 |
| MNNVL | 跨节点 NVLink 域内的 GPU 访问与内存交换 | 不经过 RDMA 网卡路径，但依赖 IMEX 与 fabric 域健康 |
| NVLS（NVLink SHARP） | NVSwitch 域内的归约与多播能力 | 是 NVSwitch / NVLink 域内的集合通信优化，不等同于 IB SHARP，也不等同于 MNNVL 本身 |

IB SHARP 把归约动作放到 IB 交换机上；NVLS 把归约 / 多播能力放在 NVSwitch 域内。MNNVL 使 NVLink 域可以跨节点，但它不自动等于「所有集合通信都使用 NVLS」。路径是否被选中，仍要看通信库的拓扑探测与算法选择。

## 诊断入口为什么是域，而不是单机

单机 NVSwitch 机型上，fabric 注册状态是管理通路的一部分。跨出机箱后，判断对象从「本机 fabric 是否注册成功」变成「整个域是否一致」：

| 检查对象 | 单机视角 | 多机域视角 |
| --- | --- | --- |
| fabric 状态 | 本机 GPU 是否 completed | 所有参与节点的 GPU 是否都 completed |
| UUID | 本机是否有域标识 | 所有节点是否使用同一个 UUID |
| clique | 本机属于哪个分组 | 参与通信的 rank 是否属于预期 clique |
| 句柄能力 | 本机是否支持 fabric handle | 句柄能否跨节点导出并导入 |
| 健康 | 本机 NVLink 是否有错误 | 域内任意链路或节点降级都可能改变可用带宽 |

因此，MNNVL 的失败经常不是「某台机器完全坏」，而是**域一致性被破坏**：一个节点未就绪、一个 clique 不一致、一条跨节点 NVLink 链路降级，或 IMEX 通道没有配置好。此时先修域，再谈训练作业性能。

**第一反应不要做什么**：不要因为机器上有 NVSwitch 与多机线缆就断定 NCCL 走 MNNVL；不要把 NVLS、IB SHARP 与 MNNVL 混成同一个开关；不要在 clique 或 UUID 不一致时先调集合通信算法；不要把 `NCCL_MNNVL_UUID` 与 `NCCL_MNNVL_CLIQUE_ID` 当成常规性能旋钮。

## 参考文档

- NCCL User Guide 的环境变量一页：`NCCL_MNNVL_ENABLE`、`NCCL_MNNVL_UUID`、`NCCL_MNNVL_CLIQUE_ID`、`NCCL_CUMEM_ENABLE`。
- NCCL User Guide 与源码中 MNNVL 初始化日志、IMEX 通道告警与 fabric 字段的口径。
- CUDA 文档中 IMEX domain 的说明。
- NVIDIA Fabric Manager User Guide 中多机 NVLink 域、UUID 与 clique 的管理方式。

