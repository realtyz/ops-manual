---
tags:
  - Linux
  - 可观测性
  - 日志
  - rsyslog
  - syslog
  - 日志集中
created: 2026-09-19
---

# 日志的集中：远程 syslog 与采集器

> [!cite] 参考资料
> `man 1 logger`（`--server`/`--port`/`--udp`/`--tcp`）、`man 5 rsyslog.conf`（`@@`/`@` 与 action 语法）、`man 3 syslog`（facility/severity）、RFC 5424（syslog 协议与 `timeQuality` 结构化数据）、RFC 3164（旧 BSD 格式）；Loki、Elasticsearch/Filebeat、Vector、Fluent Bit 的官方「getting started」文档。
>
> **实测环境：Ubuntu 24.04.5 LTS（VMware 虚拟机）/ 内核 `6.8.0-139-generic` / systemd 255（`255.4-1ubuntu8.17`）/ root 可用。** `logger`、`nc`（netcat-openbsd）、`socat` 都已正常安装（**不需要解包运行**）。远程报文用 `nc` 在本章分配到的端口上抓取（**UDP 20601 / TCP 20602**，本章端口区间 20600–20699），抓完即删、未留监听进程。
>
> **真正的跨机集中（两台机器之间）、接收端 `rsyslog` 的 `imudp`/`imtcp` 配置、TLS、RELP、断网重连与中心机容量均未实测**——本机只有一个 loopback，且 `rsyslog` 如果要新增接收端口就得改配置并重启 `rsyslog`，那属于本章纪律禁止的动作。这部分保持「未实测」，清单见子笔记 18 的实验 11。

> **这篇讲什么**：日志怎么离开本机、在网线上长什么样、中心机怎么接住它，以及顺带回答一个常被忽略的问题——**集中之后，容量这件事归谁管**。
>
> **必须先读什么**：[[Linux/09_日志与监控/05_日志的落点_journald与rsyslog|05 日志的落点：journald 与 rsyslog]]、[[Linux/09_日志与监控/02_前置_证据的时间与格式基线|02 前置：证据的时间与格式基线]]（报文里的时间戳与 `timeQuality`）。
>
> **读完能回答**：① 一条 syslog 报文由哪些字段组成，`<13>` 是什么意思？② UDP、TCP、TLS、RELP 该怎么选？③ rsyslog / Filebeat / Promtail / Vector 各自的适用场景是什么？④ 集中之后最容易新增什么风险？
>
> 所属：[[Linux/09_日志与监控/00_导读与知识地图|09 日志、监控与可观测性]] 的「一条日志的一生」第 4 站 · 主要练 **S4 远程日志集中**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住六句话
> - **syslog 报文是有格式的，不是「一段文本」；UDP 报文没有换行符，TCP 有。**
>   - 证据：实测 UDP 报文为 `<13>1 2026-09-19T08:08:01.379099+00:00 linux-lab labtest - - [timeQuality tzKnown="1" isSynced="0"] udp-remote-hello`（端口 20601）；`cat -A` 下 UDP 行尾直接接下一段输出（无 `$`），TCP 那一行有 `$`。
> - **`<13>` 是 PRI = facility×8 + severity。**
>   - 证据：本机实测换算 `13 = 1×8 + 5` → facility=user、severity=notice；对照 `logger` 默认优先级就是 `notice`，两者一致。
> - **`timeQuality` 直接回答了「这条日志的时间信不信得过」，而本机给出的是「不可信」。**
>   - 证据：本机报文里带 `tzKnown="1" isSynced="0"`，并且**没有 `syncAccuracy` 字段**——因为这台机器的时钟本来就没同步上（子笔记 02 实测 `System clock synchronized: no`、`Packet count: 0`）。看到 `isSynced="0"` 就别拿它做时间线推断。
> - **UDP 丢了完全无感知，TCP 丢了当场报错——这一点可以直接验证。**
>   - 证据：往**没有接收端**的端口发同一条消息，`logger --udp` 的**退出码是 `0`**（消息悄悄消失），`logger --tcp` 的**退出码是 `1`** 并打印 `logger: failed to connect to 127.0.0.1 port 20698`。
> - **`@` 是 UDP、`@@` 是 TCP；跨公网/跨机房要加 TLS 或 RELP。**
>   - 怎么验证：翻 `/etc/rsyslog.d/` 里的转发规则，看写的是 `@` 还是 `@@`，并确认「接收端地址与端口谁在维护」。
> - **集中只是把「磁盘满」从一台机器搬到中心机。**
>   - 怎么验证：中心机有没有自己的保留策略与容量水位告警；没有的话，故障只是换了发生地点。

## 1. 报文长什么样

用 `logger` 直接发往一个本机监听的端口（绕开系统 rsyslog），就能看到原始报文。**端口用本章分配到的区间（20600–20699）**：

```bash
rm -f /tmp/09a-udp.log /tmp/09a-tcp.log
( timeout 5 nc -u -l 20601 > /tmp/09a-udp.log & ); sleep 0.5
logger --server 127.0.0.1 --port 20601 --udp -t labtest "udp-remote-hello"; sleep 1
cat -A /tmp/09a-udp.log | head -2
( timeout 5 nc -l 20602 > /tmp/09a-tcp.log & ); sleep 0.5
logger --server 127.0.0.1 --port 20602 --tcp -t labtest "tcp-remote-hello"; sleep 1
cat -A /tmp/09a-tcp.log | head -2
python3 -c "p=13; print('PRI=%d -> facility=%d severity=%d' % (p, p//8, p%8))"
```

本机实测输出：

```text
=== UDP 20601 ===
<13>1 2026-09-19T08:08:01.379099+00:00 linux-lab labtest - - [timeQuality tzKnown="1" isSynced="0"] udp-remote-hello=== TCP 20602 ===
<13>1 2026-09-19T08:08:02.887870+00:00 linux-lab labtest - - [timeQuality tzKnown="1" isSynced="0"] tcp-remote-hello$
PRI=13 -> facility=1 severity=5
```

注意 `=== TCP 20602 ===` 紧贴在 UDP 报文后面、而 TCP 那行末尾有 `$`（换行符的 `cat -A` 表示法）：**这正是 UDP 与 TCP 在 syslog 里的一个实务差异——UDP 一条报文就是一个消息，没有分隔符，接收端只能靠「一包一条」来切分。**

再补一个**只能靠对照才能发现**的差异（同一个脚本里接着跑）：

```bash
logger --server 127.0.0.1 --port 20699 --udp -t labtest "udp-nobody-listening"; echo "udp logger exit=$?"
timeout 8 logger --server 127.0.0.1 --port 20698 --tcp -t labtest "tcp-nobody-listening"; echo "tcp logger exit=$?"
```

```text
udp logger exit=0
logger: failed to connect to 127.0.0.1 port 20698
tcp logger exit=1
```

**两个端口都没有接收端，UDP 却返回成功、TCP 直接失败。** 这就是「UDP 丢日志无感知」最直接的证据：发送端拿不到任何反馈，日志消失得悄无声息。

### 1.1 逐字段拆解（RFC 5424）

| 字段 | 实测值 | 含义 |
| --- | --- | --- |
| PRI | `<13>` | facility×8 + severity；`13 = 1×8 + 5` → facility=user（1）、severity=notice（5） |
| 版本 | `1` | RFC 5424 的版本号；旧 BSD 格式（RFC 3164）没有这一段 |
| 时间戳 | `2026-09-19T08:08:01.379099+00:00` | **带时区偏移**，跨时区对账的前提 |
| 主机名 | `linux-lab` | 发送方主机 |
| app-name / procid / msgid | `labtest - -` | 应用标识、进程号、消息 ID；未设置时为 `-` |
| 结构化数据 | `[timeQuality tzKnown="1" isSynced="0"]` | 时间同步质量：时区已知，但**未同步**（因此没有 `syncAccuracy`） |
| 消息体 | `udp-remote-hello` | 正文（UDP 无换行，TCP 有） |

> [!tip] 报文里的 `timeQuality` 是免费的时间基线证据
> `isSynced="1"` 表示发送方认为自己的时钟已同步，`syncAccuracy` 给出精度估计。**本机实测拿到的是 `isSynced="0"`，而且连 `syncAccuracy` 都没有**——这与子笔记 02 的「先对表再排障」是同一条纪律，也说明这个字段确实能反映真实状态：**收到一条 `isSynced="0"` 的远程日志时，不要用它做时间线推断。** 顺带得到一个采集侧的用法：在接收端按 `isSynced` 分流/打标，就能把「时间不可信的来源」筛出来。

## 2. 怎么把日志送出去

### 2.1 rsyslog 转发（最通用）

`/etc/rsyslog.d/60-forward.conf` 的最短写法（**示意，本机未在生产环境实测**）：

```ini
*.* @@loghost.example.com:514     # @@ = TCP；把 @@ 改成 @ 就是 UDP
```

同一件事也可以用现代 action 语法写——多写几个参数就能配队列与重试：

```ini
*.* action(type="omfwd" target="loghost.example.com" port="514" protocol="tcp" action.resumeRetryCount="-1")
```

**发送端最容易漏的三件事**：

1. **队列与重试**：默认内存队列在接收端不可达时会丢消息；要「断网不丢」得配磁盘队列（`queue.type="disk"`）与重试次数。
2. **限速**：rsyslog 自身有速率限制（`$SystemLogRateLimitInterval` 一类的旧指令或 `imjournal` 参数），高频时会主动丢弃。
3. **`imjournal` 与 `imuxsock`**：从 journald 转发与从套接字接收是两条不同的输入链路，配置时要分清。

### 2.2 传输方式的选择

| 方式 | 可靠性 | 加密 | 适用 |
| --- | --- | --- | --- |
| UDP（`@`） | 会丢（拥塞、接收队列满） | 无 | 同机房、量大、丢一点可接受 |
| TCP（`@@`） | 有连接与重传，但有队头阻塞与重连成本 | 无 | 同机房、要求不丢 |
| RELP | 应用层确认，断线可续传 | 可选 | 要求不丢且链路不稳定 |
| syslog over TLS | 依赖 TCP | **有** | 跨公网、跨机房、合规要求 |
| 采集器走 HTTP/TLS（Filebeat/Promtail/Vector） | 依赖 HTTP 重试 | 有 | 需要解析、过滤、多路输出 |

> [!important] UDP 的「丢」是无感知的
> 发送端不会收到任何错误——日志消失得悄无声息。**本机实测把这一点变成了可复现的数字**：向 `127.0.0.1:20699`（**没有任何进程在听**）发一条 UDP，`logger` 的退出码是 `0`；同样的情况换成 TCP（`127.0.0.1:20698`），`logger` 退出码是 `1` 并在标准错误上打印 `logger: failed to connect to 127.0.0.1 port 20698`。**只有在接收端做「发送计数 vs 接收计数」的对账，或者干脆改 TCP/TLS，才能发现丢了。** 这也是把关键审计日志走 UDP 时最大的隐患。

### 2.3 集中方案的选型

| 方案 | 传输 | 优点 | 代价与边界 |
| --- | --- | --- | --- |
| 远程 syslog（rsyslog / syslog-ng） | UDP/TCP/RELP/TLS | 系统自带、协议简单、所有设备都支持 | 结构化能力弱；UDP 会丢；轮转与保留要自己配 |
| Filebeat → Elasticsearch | TCP/TLS | 生态成熟、解析与索引能力强 | 资源占用与集群成本高 |
| Promtail → Loki | HTTP | 与 Prometheus 标签体系一致、成本可控 | 索引能力弱于 ES，适合「按标签找日志再 grep」 |
| Vector / Fluent Bit | 多种 | 轻量、可做转换/过滤/多路输出 | 要自己维护配置与规则 |

选型的最短判断路径：

1. **只需要「留存 + 偶尔 grep」** → 远程 syslog + 中心机轮转（成本最低）。
2. **需要按服务/实例/`trace_id` 检索** → Loki（与指标标签一致）或 ES（检索能力更强）。
3. **需要在采集端做解析、过滤、多路分发** → Vector / Fluent Bit。

## 3. 集中之后新增的风险

| 风险 | 表现 | 对策 |
| --- | --- | --- |
| 中心机磁盘被写满 | 一台中心机接几十上百台机器的日志，增长速度是叠加的 | 中心机也要有轮转/保留策略与容量告警（子笔记 08、14） |
| 单点故障 | 中心机挂了，发送端继续写本机但可能积压或丢弃 | 发送端配磁盘队列；接收端做冗余或至少监控 |
| 时间不统一 | 各机器时钟不同，集中后时间线更乱 | 先做时间基线（子笔记 02），报文的 `timeQuality` 可用于筛查 |
| 网络抖动丢日志 | UDP 静默丢包 | 关键日志改 TCP/TLS 或 RELP；在接收端做计数对账 |
| 权限与合规 | 日志里含敏感信息，集中后被更多人可见 | 采集端做脱敏/过滤；访问控制与留存策略写进方案 |

## 4. 生产动作：把「本机 → 中心机」这条链路走通

在实验机上（两台可快照的虚拟机最好；单机也可以用两个端口模拟）。**端口只用本章分配到的 20600–20699**：

> [!example]- 实验 3：验证「日志真的到达接收端」，并看清 UDP 与 TCP 的差别
> 目标：不依赖任何平台，用最小手段证明「发送端发出去了、接收端收到了」，并观察两种传输的差异。
> ```bash
> rm -f /tmp/09a-udp.log /tmp/09a-tcp.log
> ( timeout 5 nc -u -l 20601 > /tmp/09a-udp.log & ); sleep 0.5
> logger --server 127.0.0.1 --port 20601 --udp -t labtest "udp-remote-hello"; sleep 1
> cat -A /tmp/09a-udp.log                                  # 验证：收到报文，且没有换行结尾
> ( timeout 5 nc -l 20602 > /tmp/09a-tcp.log & ); sleep 0.5
> logger --server 127.0.0.1 --port 20602 --tcp -t labtest "tcp-remote-hello"; sleep 1
> cat -A /tmp/09a-tcp.log                                  # 验证：收到报文，以 $ 结尾（有换行）
> python3 -c "p=13; print('PRI=%d -> facility=%d severity=%d' % (p, p//8, p%8))"
> logger --server 127.0.0.1 --port 20699 --udp -t labtest "udp-nobody-listening"; echo "udp logger exit=$?"
> timeout 8 logger --server 127.0.0.1 --port 20698 --tcp -t labtest "tcp-nobody-listening"; echo "tcp logger exit=$?"
> ss -lntup | grep -E ":206[0-9][0-9]" || echo "(20600-20699 无监听残留)"
> rm -f /tmp/09a-udp.log /tmp/09a-tcp.log
> ```
> **预期**：两条报文都能抓到，内容形如 `<13>1 <带时区时间戳> <主机名> labtest - - [timeQuality …] <消息>`；UDP 无换行、TCP 有换行；PRI 换算为 facility=1、severity=5；**无接收端时 UDP 退出码 0、TCP 退出码 1**。
> 本机实测（UDP/TCP 两条报文的内容与退出码）见第 1 节，此处不重复。
> **风险**：低（只在本机 20600–20699 端口短暂监听，不影响系统 rsyslog；`nc` 由 `timeout` 自动结束）。
> **回滚**：`rm -f` 临时文件；`nc` 由 `timeout` 自动结束。本机实测跑完 `ss -lntup | grep 206` 无任何残留。
> **耗时**：10 分钟。
>
> **真正的跨机集中未实测**，原因有二：① 本机只有一个 loopback，没有第二台机器；② 让本机 `rsyslog` 充当接收端需要新增 `imudp`/`imtcp` 模块配置并**重启 `rsyslog`**，这属于本章纪律明确禁止的动作。需要配置接收端、发送端转发、验证断网重连与中心机容量的完整清单见子笔记 18 的实验 11。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 「日志集中了就安全了」 | 只是把磁盘满的风险挪到中心机；中心机同样需要保留策略与容量告警 |
| 「用 UDP 更快，反正日志丢了也不重要」 | UDP 丢包**完全无感知**；本机实测向无接收端的 UDP 端口发送时 `logger` 退出码仍是 `0`，审计与合规类日志走 UDP 是隐患 |
| 「配了转发就一定不丢」 | rsyslog 默认内存队列在接收端不可达时会丢；要配磁盘队列与重试。本机的对照实验说明**连接阶段** TCP 至少会报错，但已建连后的队列满、断链期间 TCP 同样会丢 |
| 「报文里时间戳都有，不用管时钟」 | 报文时间来自发送方时钟；本机实测报文里就是 `isSynced="0"`（因为本机时钟没同步上），这是明确的「不可信」信号 |
| 「中心机先收着，容量以后再说」 | 中心机增长速度是各来源叠加，满盘往往来得比预期快 |
| 「接入平台就等于能查」 | 若没带实例/服务字段，集中之后仍然只能按时间盲搜（子笔记 04） |

## 决策练习

**场景**：公司要把 200 台服务器的 `/var/log/auth.log` 集中到一台中心机，用于安全审计，保留 180 天。有人在方案里写了「用 UDP 514，简单、性能好」。

**A. 同意，UDP 性能最好，日志量也不大**
**B. 改成 TCP/TLS（或 RELP），并在中心机配保留策略、容量水位与接收计数对账**
**C. 干脆不集中，需要时逐台登录去看**

**为什么选 B**：需求里有三个关键词——**安全审计**（丢日志是合规问题）、**200 台**（中心机容量是主要风险）、**180 天**（保留策略必须显式设计）。TCP/TLS 解决可靠性，保留与容量解决中心机风险，计数对账解决「怎么知道丢了」。

**为什么不选 A**：UDP 的丢包无感知，对审计场景是硬伤；「日志量不大」是拍脑袋结论——200 台认证日志的实际量要在采集端量过才知道。

**为什么不选 C**：逐台登录查在 200 台规模下不可行，而且不符合审计的「可集中检索、可长期留存」要求。

## 要点自测

> [!question]- 一条 RFC 5424 syslog 报文包含哪些字段？`<13>` 是什么意思？
> - **结构**：`<PRI>版本 时间戳 主机名 应用名 进程号 消息ID 结构化数据 消息体`。
> - **`<13>`**：PRI = facility×8 + severity；`13 = 1×8 + 5` → facility=user、severity=notice（本机实测换算）。
> - **时间戳带时区**（实测 `+00:00`，因为本机时区是 UTC），这是跨时区对账的前提。
> - **结构化数据**里实测带 `timeQuality`（`tzKnown="1" isSynced="0"`，本机没同步上所以没有 `syncAccuracy`）。
> - **第一反应不要是什么**：不要把报文当成「一段文本」直接丢掉结构化数据。

> [!question]- UDP、TCP、TLS、RELP 应该怎么选？
> - **UDP（`@`）**：快、无连接、**会静默丢包**；本机实测向无接收端的端口发 UDP，`logger` 退出码仍是 `0`。只适合同机房且容忍丢失的场景。
> - **TCP（`@@`）**：有重传，但有队头阻塞与重连成本；同机房要求不丢时可接受。本机实测向无接收端的端口发 TCP，`logger` 退出码 `1` 并报 `failed to connect`——**至少在连接建立阶段是「会报错」的**。
> - **RELP**：应用层确认、断线可续传，适合链路不稳定但要求不丢。
> - **syslog over TLS**：跨公网/跨机房/合规场景的默认选择（本机未实测）。
> - **第一反应不要是什么**：不要把审计类日志配成 UDP。

> [!question]- 日志集中之后新增了哪些风险？
> - **中心机容量**：增长是各来源叠加，必须配保留策略与容量告警。
> - **单点故障**：接收端挂掉会导致积压或丢弃，发送端要有磁盘队列。
> - **时间不统一**：集中后更容易把不同机器的证据拼在一起，先做时间基线。
> - **权限与合规**：日志集中后可见面变大，需要访问控制与脱敏。
> - **第一反应不要是什么**：不要以为集中等于「更安全、更省事」。

> 上一篇：[[Linux/09_日志与监控/06_日志的轮转_logrotate的两种切法|06 日志的轮转：logrotate 的两种切法]] ｜ 下一篇：[[Linux/09_日志与监控/08_日志的退役_容量治理与磁盘写满|08 日志的退役：容量治理与磁盘写满]]
