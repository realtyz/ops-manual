---
tags:
  - AI-Infra
  - InfiniBand
  - fabric
  - 实操
created: 2026-10-11
---

# fabric 级体检

fabric 级体检回答的不是「这台机器能不能上网」，而是「这张 fabric 在当前时间窗内，哪些 rail、端口、成员关系或域状态不健康，影响面是什么」。因此开始前必须先固定三个对象：**时间窗、拓扑范围、预期基线**。没有这三项，读到的错误计数只能说明「历史上发生过错误」，不能回答这次作业异常由谁造成。

## 体检前的三个输入

| 输入 | 内容 | 缺失时的后果 |
| --- | --- | --- |
| 时间窗 | 作业异常开始与结束时间，前后各留出收敛与采样余量 | 计数器差分无法与事件对齐 |
| 拓扑范围 | 参与作业的节点、网卡、rail / plane、叶与脊交换机 | 无法判断影响面，只能整网模糊排查 |
| 预期基线 | golden topology、端口速率、成员关系、域状态、历史错误水位 | 无法判断当前状态是否偏离交付基线 |

体检过程中不要清零计数器。清零会破坏累计值的历史连续性，使差分和趋势失去口径；需要重新开始观察时，记录新的采样时间即可。

## 第一步：固定主机侧端口状态

先在每个参与节点上确认 RDMA 设备与端口。下面是只读命令：

```bash
ibstat
ibv_devinfo -v
```

`ibstat` 用于确认设备、端口状态、速率与 MTU；`ibv_devinfo -v` 用于确认 RDMA 设备、端口与能力信息。字段名在不同驱动版本中略有差异，但判据不变：

| 字段 | 判据 | 与谁同看 |
| --- | --- | --- |
| 端口状态 | 应为 active，而不是 down、init 或 unknown | 交换机侧对应端口状态 |
| 链路速率 | 应与交付基线一致 | golden topology 与网卡 / 交换机能力 |
| MTU | 应与 fabric 与主机侧配置一致 | 网卡端口状态与网络侧配置 |
| 设备名 | 应与通信库日志中的 HCA 一致 | `NCCL_IB_HCA` 与拓扑文件 |

如果主机侧端口已经 down，先不要进入整网诊断；如果端口 active 但速率低于基线，链路降速本身就是第一嫌疑。

## 第二步：读取 SM 与 fabric 拓扑状态

IB fabric 上先读 SM 状态：

```bash
sminfo
```

`sminfo` 输出子网管理器的状态信息。需要记录的判据包括：

| 字段 | 是什么 | 判据 |
| --- | --- | --- |
| SM 状态 | 当前 SM 是否处于 master / standby 等角色 | 生产时间内应处于稳定 master 状态 |
| SM 活动 | SM 是否在响应 | 查询失败或状态抖动时，主机侧计数不宜直接归因于硬件 |
| 子网信息 | 当前 fabric 的管理视图 | 应与预期 fabric 范围一致 |

再读整网拓扑与链路视图：

```bash
ibdiagnet
iblinkinfo
ibnetdiscover
```

三者分工不同：

| 工具 | 主要回答 | 适合什么时候用 |
| --- | --- | --- |
| `ibdiagnet` | fabric 级诊断、错误扫描与整体健康 | 需要整网体检时 |
| `iblinkinfo` | 每条链路的速率、状态与对端信息 | 需要核对 port-to-port 映射时 |
| `ibnetdiscover` | 拓扑发现结果 | 需要与 golden topology 做差异比较时 |

拓扑比较要落到这个映射链：

```text
node → NIC → port → rail / plane → leaf port → spine path
```

| 差异类型 | 含义 | 下一步 |
| --- | --- | --- |
| 端口缺失 | 物理链路或发现结果缺失 | 先核对交换机端口与线缆 |
| 速率不一致 | 链路降速或配置能力不同 | 转入链路质量与模块检查 |
| rail 映射不一致 | 接线与基线不符 | 核对变更记录，判断是否未批准变更 |
| 对端不匹配 | 两端看到的链路身份不同 | 以两侧状态与拓扑发现结果共同判断 |

## 第三步：读端口错误与流量计数

端口计数器是累计值，必须取差分：

```bash
ibqueryerrors
perfquery
```

`ibqueryerrors` 汇总错误；`perfquery` 读取端口性能计数。常见读数包括 `SymbolError`、`LocalLinkIntegrityErrors`、`PortRcvErrors`、`PortXmitDiscards`、`PortXmitData`、`PortRcvData` 一类字段；具体名称以现场工具版本为准。

| 字段类型 | 单位 / 口径 | 判据 | 与谁同看 |
| --- | --- | --- | --- |
| 误码类计数 | 累计次数 | 同一时间窗内增长率异常，且集中在某个端口或 rail | 交换机侧同链路错误计数 |
| 丢弃类计数 | 累计次数 | 增长说明端口或队列丢弃报文 | 队列与拥塞标记 |
| 流量类计数 | 累计量，计数单位为 4 字节 | 取差分并与预期 rail 分布比较 | 通信库选路结果 |
| 链路断开类计数 | 累计次数 | 任何增长都值得与作业时间线对齐 | SM / 交换机链路事件 |

读数时必须回答四个问题：

1. 这个计数属于哪个 node / NIC / rail / switch port；
2. 两次采样的时间点是什么；
3. 这个时间窗内是否有 SM、路由、链路或作业事件；
4. 对端同一链路的计数是否也能解释。

## 第四步：核对成员关系

成员关系的证据在 SM 或交换机配置里，不在单机 GPU 日志里。IB 上核对 PKey，RoCE 上核对 VLAN / VRF 与路由可达性。

| 核对对象 | 要回答的问题 | 与谁同看 |
| --- | --- | --- |
| PKey 表 | 发端与收端端口是否属于同一分区 | 通信库建链报错、失败 rank 集合 |
| VLAN / trunk | 两端是否在同一二层域，或经预期 trunk 互通 | 网络设备状态与地址信息 |
| VRF / 路由表 | 地址选择是否落在预期路由域 | 主机侧路由与 GID 表 |
| 作业网卡清单 | 作业实际使用的 NIC 是否都在预期成员关系内 | `NCCL_IB_HCA` 与通信库日志 |

成员关系错误的第一特征是**失败集合有结构性**：失败 rank 往往集中在某些节点、某些网卡或某些地址段，而不是随机分布。把失败集合映射回 node / NIC / rail / port 后，再与成员表重叠，可以快速判断是否属于这一类问题。

## 第五步：核对多机 NVLink 域

对 MNNVL 参与节点，先读 GPU fabric 与域信息。NCCL 的告警会把 IMEX 问题指向 `/dev/nvidia-caps-imex-channels` 与 `nvidia-imex-ctl` 一类 IMEX 配置入口：

```bash
nvidia-smi -q
```

`nvidia-smi -q` 在支持的机型上会给出 GPU fabric 相关状态；IMEX 通道的检查与配置由 `nvidia-imex-ctl` 承担，其子命令以现场工具版本的帮助信息为准。两类工具的输出形态与平台版本相关，但核对对象不变：

| 对象 | 判据 | 与谁同看 |
| --- | --- | --- |
| GPU fabric 状态 | 参与节点均应为完成 / successful 状态 | Fabric Manager 日志与 NCCL MNNVL 日志 |
| fabric UUID | 同一 MNNVL 域内应一致 | 每个节点的域报告 |
| clique ID | 参与同一通信组的 rank 应属于预期 clique | 作业 rank 分布与 NCCL 日志 |
| healthMask / degraded bandwidth | 不应报告降级带宽 | NVLink 链路状态与 GPU fabric 健康 |
| IMEX 通道 | 通道存在且配置可用 | NCCL 的 MNNVL 告警 |

这一步的结论只有两种有效写法：

- **域健康且 NCCL 使用了 MNNVL**：需要同时有域状态一致与 NCCL 日志中的 MNNVL 字段；
- **域不健康或未使用**：指出具体缺口是状态、UUID、clique、healthMask 还是 IMEX 通道，而不是只写「MNNVL 有问题」。

## 现场定位表

| 现象 | 先读什么 | 转向哪一步 |
| --- | --- | --- |
| 单节点网卡 down | `ibstat`、交换机端口状态 | 先查该链路，不做整网结论 |
| 多节点同时超时 | `sminfo`、SM / 路由事件时间线、两侧计数差分 | 判断是否处于控制面收敛窗口 |
| 一条 rail 反复异常 | `ibqueryerrors`、`iblinkinfo`、golden topology | 定位该 rail 的端口、线缆与模块 |
| 部分 rank 建链失败 | 失败 rank 映射、PKey / VLAN / VRF | 先修成员关系 |
| 多 plane 只有一个有流量 | 通信库日志中的 plane 字段、各 plane 链路计数 | 核对 `NCCL_IB_HCA` 与 rail / plane 分配 |
| MNNVL 能力未启用 | NCCL MNNVL 字段、GPU fabric 状态、IMEX 通道 | 先修域，再测通信性能 |
| 作业慢但无错误 | rail / plane 流量分布、队列与拥塞计数、同口径基线 | 转入性能对照实验 |

## 体检产物

一次可复盘的 fabric 体检至少应保存：

| 产物 | 内容 |
| --- | --- |
| 时间窗 | 采样开始、结束与作业异常时间 |
| 节点清单 | 参与节点、rank、GPU 与 NIC 映射 |
| 拓扑快照 | 当前发现结果与 golden topology 差异 |
| 端口状态 | 主机侧与交换机侧端口状态、速率、成员关系 |
| 计数差分 | 同一时间窗内错误、丢弃、流量与拥塞计数增量 |
| fabric 事件 | SM、路由、链路、分区与域状态事件 |
| 结论 | 明确影响面、证据链与下一步责任人 |

**第一反应不要做什么**：不要在多机故障中只用单机命令下结论；不要清零计数器；不要把 SM 收敛窗口内的超时直接当成硬件故障；不要没有 golden topology 就声称接线错误；不要把 MNNVL 硬件存在当成路径已经使用的证据。

## 参考文档

- `ibstat`、`ibv_devinfo`、`sminfo`、`ibdiagnet`、`iblinkinfo`、`ibnetdiscover`、`ibqueryerrors`、`perfquery` 的手册页。
- InfiniBand Architecture 中端口计数器、SM 与分区的定义。
- UFM 文档中拓扑基线、事件与 fabric 诊断的口径。
- CUDA 文档中 IMEX domain 的说明，以及 NVIDIA Fabric Manager User Guide 中多机 NVLink 域的管理方式。

