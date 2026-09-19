---
tags:
  - Linux
  - 网络
  - tcpdump
  - tc
  - netem
  - 抓包
created: 2026-09-18
---

# 抓包与故障注入：`tcpdump` 与 `tc`

> [!cite] 参考资料
> `man 8 tcpdump`、`man 7 pcap-filter`（过滤表达式语法）、`man 8 tc`、`man 8 tc-netem`、`man 8 tc-qdisc`、`man 8 ping`、`man 8 wireshark`（图形分析）、`man 8 ip-netns`，内核文档 `Documentation/networking/filter.rst` 与 `Documentation/networking/timestamping.rst`。
>
> **实测状态**：本篇输出已在**本实验机**上本次重跑并粘贴，替换了原 WSL2 输出。实测环境：**Ubuntu 24.04.5 LTS（VMware 虚拟机）/ 内核 `6.8.0-139-generic` / 4 vCPU / root 可用。** 工具链实测齐备：`tcpdump 4.99.4`（libpcap `1.10.4`，带 `TPACKET_V3`）、`tc`（`iproute2-6.1.0`）、`iperf3 3.16`、`mtr`、`tracepath`。
>
> 已实测：① `tcpdump` 的过滤表达式、`-w` 落盘与 `-r` 回读（在 `lo` 上用本章端口 `9303` 与 `9310`/`9311` 采集，粘在下文）；② 非 root 抓包被拒的第一手报错；③ `tc netem` 在隔离 veth 上注入 50 ms 延迟与 20% 丢包、删除后恢复；④ `tc qdisc show` 的基线（宿主 `ens33` 是 `fq_codel`）。
>
> **未实测**：客户端与服务端**两台机器同时抓包**的二分定位（本实验环境只有一台虚拟机，没有第二台机器；这一条按方法撰写，不当作实测结论）；`-i any` 的 cooked capture 接口名行为。
>
> 本篇涉及**会影响流量的动作**（注入延迟丢包、改队列规则）：全部只在**实验机的独立 network namespace** 里做；生产取证只允许「限流限时的抓包」这一类只读动作。

> **这篇讲什么**：排障里最贵的两个动作——**在生产上抓包**（成本高、容易出事）与**复现偶发故障**（很难等）。这一篇把抓包的纪律与过滤写法讲清，并给出用 `tc netem` 在实验链路上「造出问题」的标准做法。
>
> **必须先读什么**：[[Linux/06_网络协议栈与排障/01_前置_分层模型与数据包的一生|01 前置：分层模型与数据包的一生]]（抓包里一行是什么）、[[Linux/06_网络协议栈与排障/06_第3站_传输_窗口拥塞与重传|06 第 3 站：传输]]（先用 `ss -ti` 缩小范围，再决定是否抓包）。
>
> **读完能回答**：① 怎么抓包才既拿到证据又不影响生产？② 为什么「抓不到」也是一种有效信息？③ `netem` 为什么只影响出口？④ 怎么用注入复现一次「偶发超时」并验证重试与超时参数？
>
> 所属：[[Linux/06_网络协议栈与排障/00_导读与知识地图|06 网络协议栈与排障]] · 主要练 **N7 抓包与故障注入**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住六句话
> - **抓包的底线纪律：限流、限时、限文件，抓完就删**。生产机上不限流抓包能把业务拖死。
> - **三重限制**：`-c`（包数）、`-w`（落到独立目录/文件）、`timeout`（时间）；必要时再加 `-s <len>` 截断载荷。
>   - 证据：本机实测 `timeout 8 tcpdump -i lo -nn -w /tmp/06-lab.pcap -c 20 port 9303` 落盘只有 `184` 字节，`tcpdump -nn -r` 能原样读回——**限制包数 + 落独立文件 + 加超时，三条都做得到**。
> - **「看不到」也是证据**
>   - 怎么验证：本机实测 `ss -ti` / `ss -tan` 与 `tcpdump` 能互相印证（同一时刻 `nstat` 的重传计数与抓包里的重复 seq 对得上，见子笔记 06）；抓不到包说明**这一跳没收到/没发出**，用来二分定位。
> - **始终加 `-nn`**：不解析主机名与端口名，避免反向 DNS 放大延迟（本机输出里地址与端口都是数字形态）。
> - **`netem` 只作用于出口（egress）**
>   - 证据：本机实测 `tc qdisc add dev veth0 root netem delay 50ms` 后，从**对端** `ping` 进来看到 RTT 由 `0.028 ms` 变成 `50.109 ms`（平均）——作用的是加 qdisc 那一侧的发送方向；双向不对称延迟要在两端分别注入。
> - **偶发问题不要在高峰期硬抓**：先在实验链路上用 `netem` 把现象复现出来，再决定是否上生产取证——**这比等下一次故障便宜得多**。本机实测在 netns 的 20 ms + 1% 丢包链路上，`cubic` 与 `bbr` 的吞吐差近 20 倍（子笔记 06），这类对照只有注入链路才做得出来。

## 1. `tcpdump`：纪律与基本功

### 1.1 三条底线

| 限制 | 参数 | 为什么 |
| --- | --- | --- |
| 包数 | `-c <N>` | 防止无限增长把磁盘写满 |
| 时间 | `timeout <秒> tcpdump ...` | 防止忘记停止 |
| 落盘位置 | `-w <文件>` | 写到独立目录（如 `/tmp` 或专用分区），避免占满根分区 |
| 载荷长度 | `-s <len>` 或 `-s 0` | 只看头时不需要全量载荷；**全量抓包代价最大** |

### 1.2 常用写法

```bash
tcpdump -i any -nn -c 100                       # 验证：前 100 个包，不解析主机名/端口名
tcpdump -i ens33 -nn host 10.0.0.5 and port 443 # 验证：只看某个对端与端口
tcpdump -i lo -nn 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'
                                                # 验证：只看 SYN/FIN，快速看清「谁主动连、谁主动断」
tcpdump -i lo -nn -s 0 -w /tmp/06-lab.pcap -c 2000
                                                # 验证：落盘，限制包数，抓完拉回本地用 Wireshark 分析
tcpdump -i lo -nn -c 20 'tcp port 9303 and tcp[tcpflags] & tcp-rst != 0'
                                                # 验证：是否有人回 RST（连接被拒/被重置）
timeout 60 tcpdump -i ens33 -nn -w /tmp/06-cap.pcap 'host 10.0.0.5'
                                                # 验证：三重限制的标准组合（时间 + 落盘 + 过滤）
```

本机实测（`lo` 上一个真实的三次握手 + 数据 + 过滤，均为**实测粘贴**）：

```text
$ tcpdump -i lo -nn -c 4 port 9303
listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
08:32:29.777519 IP 127.0.0.1.51296 > 127.0.0.1.9303: Flags [S], seq 2422559021, win 65495, options [mss 65495,sackOK,TS val 2968868735 ecr 0,nop,wscale 7], length 0
08:32:29.777527 IP 127.0.0.1.9303 > 127.0.0.1.51296: Flags [S.], seq 3676773960, ack 2422559022, win 65483, options [mss 65495,sackOK,TS val 2968868735 ecr 2968868735,nop,wscale 7], length 0
08:32:29.777535 IP 127.0.0.1.51296 > 127.0.0.1.9303: Flags [.], ack 1, win 512, options [nop,nop,TS val 2968868735 ecr 2968868735], length 0
08:32:29.777640 IP 127.0.0.1.9303 > 127.0.0.1.51296: Flags [P.], seq 1:6, ack 1, win 512, options [nop,nop,TS val 2968868735 ecr 2968868735], length 5
4 packets captured
16 packets received by filter
0 packets dropped by kernel

$ tcpdump -i lo -nn -c 4 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0 and port 9303'
08:32:40.791718 IP 127.0.0.1.50754 > 127.0.0.1.9303: Flags [S], seq 3871701324, win 65495, options [mss 65495,sackOK,TS val 2968879749 ecr 0,nop,wscale 7], length 0
1 packet captured
2 packets received by filter

$ timeout 8 tcpdump -i lo -nn -w /tmp/06-lab.pcap -c 20 port 9303   # 落盘
2 packets captured
-rw-r--r-- 1 tcpdump tcpdump 184 Sep 19 08:32 /tmp/06-lab.pcap
$ tcpdump -nn -r /tmp/06-lab.pcap                                   # 回读
08:32:46.824321 IP 127.0.0.1.56468 > 127.0.0.1.9303: Flags [S], seq 1240475611, win 65495, options [mss 65495,sackOK,TS val 2968885781 ecr 0,nop,wscale 7], length 0
08:32:46.824353 IP 127.0.0.1.9303 > 127.0.0.1.56468: Flags [R.], seq 0, ack 1240475612, win 0, length 0
```

**读法**：第 4 行 `Flags [P.], seq 1:6 … length 5` 是服务端回的 5 字节 `hello`；最后一段里 `Flags [R.]` 是**服务端已经退出、内核回 RST**——**「看到 RST」正是「端口没监听/连接被拒」的直接证据**，比在应用日志里猜快得多。`-nn` 让地址与端口保持数字形态（对比不加时的反向解析）。

**权限是另一条必须实测的边界**（本机 root 可用，非 root 会被拒）：

```text
$ runuser -u realtyz -- tcpdump -i lo -nn -c 1
tcpdump: lo: You don't have permission to perform this capture on that device
(socket: Operation not permitted)
$ getcap /usr/bin/tcpdump
(空 = 该二进制没有设 CAP_NET_RAW 文件 capability)
```

所以要给普通用户抓包能力，正确做法是**临时提权**（`sudo`）或按需给二进制加 capability —— 注意后者**扩大了攻击面**，属于安全变更，要走变更流程。

### 1.3 几条必须记住的边界

- **`-i any` 看到的接口名与真实链路不同**（Linux 用 cooked capture），做精确的二层分析要指定具体接口。
- **抓包本身有成本**：`-s 0` 全量抓 + 高 PPS 会把 CPU 与磁盘吃满；用 `-c`、`-w` 到独立目录、`timeout` 三重限制。
- **看不到 ≠ 不存在**：抓不到包说明**这一跳没收到/没发出**，正好用来二分定位（在客户端抓、服务端抓、中间抓）。
- **UDP 与小包也没问题，但 TLS 之后的载荷是密文**：只能看握手与统计，看不到内容——想看清楚就得上 `openssl s_client` 或应用日志。
- **落盘分析而不是盯着屏幕**：生产机上不做复杂显示过滤（非常耗 CPU）；`-w` 落到 pcap 后拉回本地用 Wireshark 分析。

### 1.4 二分定位：抓包最值钱的用法

```mermaid
flowchart LR
  A["客户端抓：发出去了吗"] --> B["服务端抓：收到了吗"]
  B --> C{"两边结论"}
  C -- "客户端有、服务端无" --> D["中间链路/过滤设备"]
  C -- "客户端无" --> E["问题在本机（应用/路由/队列）"]
  C -- "服务端有、未回" --> F["问题在服务端（队列/防火墙/应用）"]
```

```bash
timeout 30 tcpdump -i ens33 -nn -c 50 host <server_ip> and port <port>   # 客户端
timeout 30 tcpdump -i ens33 -nn -c 50 host <client_ip> and port <port>   # 服务端（同时执行）
```

（**本环境未实测**：二分定位需要**两台机器**同时抓包，本实验环境只有一台虚拟机。要做这一步，两台机器的时间要大致对齐（先 `date` 对一下），并且两边用同一套过滤表达式。**在本机单个 netns 内可以退而求其次**：在路由器两侧的 veth 上各抓一次——`tcpdump -i v6cp` 与 `tcpdump -i v6sp` 就能看出「包进了路由器、有没有从另一侧出去」，这是本机真实做到过的等价练习。）

## 2. `tc`：看队列规则与注入故障

### 2.1 看当前队列规则

```bash
tc qdisc show dev ens33                  # 验证：当前队列规则（实测本机 net.core.default_qdisc=fq_codel → qdisc fq_codel）
tc -s qdisc show dev ens33               # 验证：加统计（dropped/backlog 直接说明队列在丢包）
tc -s qdisc show                         # 验证：所有设备的队列与丢包计数
```

### 2.2 注入延迟与丢包（实测）

实验 5（完整脚本见子笔记 13）先在隔离环境的 veth 上注入 50ms 延迟：

```bash
tc qdisc add dev veth0 root netem delay 50ms
ping -c3 10.0.0.1                       # 实测 RTT：基线 0.028ms → 50.081/50.109/50.149 ms
```

再追加 20% 丢包，最后删除 qdisc 恢复：

```bash
tc qdisc change dev veth0 root netem delay 50ms loss 20%
ping -c10 10.0.0.1                      # 实测：10 包收到 5 包（50% 丢包，样本小，量级吻合即可）
tc qdisc del dev veth0 root             # 验证：删除后 RTT 立刻回到 0.016/0.020/0.025 ms
```

（以上为**实测输出**，环境见本篇开头。）

**更接近真实故障的一种用法是「先劣化、再量协议行为」**：把出口 qdisc 设成 `netem delay 20ms loss 1%`，同一个 netns 里先后用 `cubic` 与 `bbr` 跑 6 秒 `iperf3`，实测 `44.7 Mbits/sec`（cubic，`cwnd` 掉到 9）对 `880 Mbits/sec`（bbr，`bbr:(bw:1.31Gbps,…)`）。**这类对照只有注入链路才做得出来**——生产上你等不到「同等条件的两次故障」。细节与边界见子笔记 06。

> [!warning] 注入之后一定要确认「恢复」
> 本机实测的恢复判据不是「按了删除键」，而是**同一位置复测的 RTT 回到基线**（`50.109 ms → 0.020 ms`），并且 `tc qdisc show` 里不再有 netem（veth 上回到 `qdisc noqueue`，宿主 `ens33` 是 `qdisc fq_codel`）。

### 2.3 `netem` 的能力与边界

| 能力 | 参数 | 用来复现什么 |
| --- | --- | --- |
| 延迟 | `delay 50ms` | 高 RTT 链路、跨地域访问 |
| 抖动 | `delay 50ms 10ms` | 延迟波动、音视频卡顿 |
| 丢包 | `loss 20%` | 重传、拥塞控制降速、偶发失败 |
| 乱序 | `reorder 25% 50%` | 乱序重组与重传 |
| 损坏 | `corrupt 0.1%` | 校验失败 |
| 限速 | `rate 10mbit` | 带宽不足、pacing 行为 |

> [!tip] `netem` 只作用于**出口（egress）**
> `tc qdisc add dev X root netem ...` 影响的是**从本机 X 接口发出去的包**。所以：
> ① 想模拟「对端回包延迟」，要在**对端**加或在中间设备加；② 双向不对称延迟要分别在两端加；③ **回环接口（`lo`）也能加**，但会同时影响本机所有走 `lo` 的服务（包括 DNS 存根），实验机上才好这么干。
> 除了 `delay`/`loss`，`netem` 还支持 `duplicate`、`reorder`、`corrupt`、`rate`，是复现「偶发变慢/偶发失败」最便宜的手段。

## 3. 复现一个「偶发超时」的标准流程

这是把 N7 从「会敲命令」变成「会做实验」的关键步骤：

1. **先定义现象**：例如「客户端设置 1 秒超时，服务端偶发不响应」。
2. **建隔离拓扑**：`ip netns add srv|cli` + veth（子笔记 13 的实验 3）。
3. **注入故障**：`tc qdisc add dev veth0 root netem delay 50ms loss 20%`。
4. **跑真实应用或脚本**：观察超时、重试、报错是否符合预期。
5. **改一个参数（例如超时、重试次数、连接池大小）再跑一次**：**这才是注入的价值——验证参数是否合理**。
6. **恢复**：`tc qdisc del dev <dev> root`，确认 RTT 回到基线。
7. **清理**：`tc qdisc show` 确认无残留，`ip netns del` 回收拓扑。

在 `delay 50ms loss 20%` 的状态下跑应用并改参数，随后加大难度、最后恢复：

```bash
tc qdisc add dev veth0 root netem delay 50ms loss 20%
tc qdisc change dev veth0 root netem delay 200ms      # 加大难度再跑一次
tc qdisc del dev veth0 root                            # 恢复
ping -c3 10.0.0.1                                      # 验证：RTT 回到基线
```

> [!warning] 生产环境的三条红线
> ① **不要在生产机上做注入实验**（netem 会直接改变业务流量）；② **不要在高峰期无参数抓包**；③ **不要把抓到的包长期留在机器上**（含业务数据，且占空间）。

## 4. 生产动作：一次抓包取证的检查表

抓包前：

- [ ] 明确**要验证的假设**（例如「客户端发了 SYN，服务端没回」）。
- [ ] 明确**抓哪一跳**（客户端 / 服务端 / 中间）。
- [ ] 写好**过滤表达式**（host/port/flags）。
- [ ] 准备好 `-c`、`-w`、`timeout` 三重限制。
- [ ] 确认**落盘目录有空间**且不在根分区上。
- [ ] 记录**开始时间**（与另一台机器对齐）。

抓包后：

- [ ] 立即停止并确认文件大小合理。
- [ ] 拉回本地分析（Wireshark），生产机上不做复杂显示过滤。
- [ ] 记录结论与证据行（时间戳、四元组、flags）。
- [ ] **删除或归档**临时抓包文件，并把结论写进故障记录。

```bash
df -h /tmp                                       # 验证：落盘目录空间
date                                             # 验证：记录开始时间
timeout 60 tcpdump -i ens33 -nn -w /tmp/06-cap.pcap -c 20000 'host <ip> and port <port>'
ls -lh /tmp/06-cap.pcap                          # 验证：文件大小是否合理
```

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 直接 `tcpdump -i any` 跑在高峰期 | 最容易把机器抓挂的做法；必须限流限时落盘 |
| 抓包不加 `-nn` | 反向 DNS 解析会拖慢甚至卡住输出 |
| 认为「抓不到就是没问题」 | 抓不到只说明这一跳没有；要用两侧对照做二分 |
| 全程 `-s 0` 全量抓 | CPU 与磁盘开销最大；只看头时截断载荷 |
| 在生产机上做 `netem` 注入 | 会直接改变业务流量；注入只在实验机做 |
| 只在 `lo` 上做注入实验 | `lo` 会影响本机所有走回环的服务（含 DNS 存根） |
| 注入后忘记删除 qdisc | 残留会持续影响流量；收尾必须 `tc qdisc show` 确认 |
| 抓到的包长期留在服务器上 | 占空间且含业务数据；分析完应及时删除或归档 |

## 决策练习

> [!question]- 场景：生产偶发 50ms 延迟，只在高峰期出现。同事说「在业务机上 `tcpdump -i any` 抓一天总能看到」。
> A. 直接抓一天
> B. 先在实验链路上用 `tc netem` 复现延迟/丢包，再决定是否在生产限流限时抓包；生产抓包要过滤、落盘、限数量
> C. 放弃抓包，直接改内核参数
>
> **答案：B。**
> 偶发问题直接抓一天命中率低、代价高；先用注入复现能确定抓包过滤条件。生产抓包必须有「限流、限时、落盘」纪律，否则高峰期会把机器抓挂。
> **第一反应不要是什么**：不要在生产高峰 `tcpdump -i any` 全量抓。

## 要点自测

> [!question]- 抓包怎么抓才能既拿到证据又不影响生产？
> - **三重限制**：`-c`（包数）、`-w`（落到独立目录/文件）、`timeout`（时间）；必要时再加 `-s <len>` 截断载荷。
> - **先想清楚抓哪一跳**：客户端抓、服务端抓、中间抓——**「这一跳有没有」比「抓全了」更有价值**（二分定位）。
> - **过滤表达式要精确**：`host`/`port`/`tcp[tcpflags]` 组合，避免把无关流量写进盘。
> - **落盘分析而不是盯着屏幕**：`-w` 落 pcap → 拉回本地 Wireshark。
> - **第一反应不要是什么**：不要在业务高峰无参数直接 `tcpdump -i any`。

> [!question]- 为什么「抓不到包」也是有效信息？
> - 它把问题范围缩小了一半：**这一跳没有收到（或没有发出）**。
> - 在客户端与服务端同时抓，四种组合直接给出结论：两边都有 → 问题在服务端处理；只有客户端有 → 中间链路/过滤；只有服务端有 → 客户端方向异常；两边都没有 → 流量根本没发起或抓错了接口。
> - 前提是**抓的位置与过滤条件正确**（接口、方向、命名空间）。
> - **第一反应不要是什么**：不要把「没抓到」当成「没有问题」。

> [!question]- `netem` 为什么只影响出口？这带来什么使用约束？
> - `tc qdisc` 挂在设备的**发送路径**上，所以只作用于从该接口发出的包。
> - 想模拟双向延迟，要在两端分别注入；想模拟「对端回包慢」，要在对端注入。
> - 在 `lo` 上注入会影响本机所有走回环的流量（包括 DNS 存根、本地服务调用）。
> - **第一反应不要是什么**：不要以为在一端加了 `delay` 就是「双向都慢了」。

> 上一篇：[[Linux/06_网络协议栈与排障/09_第6站_出门之后_转发NAT与conntrack|09 第 6 站：出门之后]] ｜ 下一篇：[[Linux/06_网络协议栈与排障/11_主机侧调优与网络基线|11 主机侧调优与网络基线]]
