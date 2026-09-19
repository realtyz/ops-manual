---
tags:
  - Linux
  - 可观测性
  - 监控
  - Prometheus
  - PromQL
  - 告警
created: 2026-09-19
---

# 指标的判定：PromQL 与告警规则

> [!cite] 参考资料
> Prometheus 官方文档的「Data model」「Querying basics」「Operators」「Alerting rules」「Recording rules」「Configuration」；`promtool check rules` 的用法。
>
> 实测输出来自 Ubuntu 24.04.4（WSL2）/ systemd 255 / 非 root：`prometheus` 以 `apt-get download` + `dpkg-deb -x` **解包运行**，抓取解包运行的 node_exporter（`127.0.0.1:19100`），规则文件写在一个必然触发的告警表达式上；实验后进程与临时目录已清理。

> **这篇讲什么**：从「采到数据」到「判定出事了」。包括 Prometheus 的数据模型、四类最常用的 PromQL 写法、记录规则与告警规则的分工，以及怎么**在不依赖通知渠道**的前提下验证一条告警规则是否真的生效。
>
> **必须先读什么**：[[Linux/09_日志与监控/09_指标的计量_node_exporter与指标语义|09 指标的计量：exporter 与指标语义]]（先有指标，再谈表达式）。
>
> **读完能回答**：① 时间序列由什么唯一确定？② `rate()` 的窗口该怎么取值、为什么不能用裸 counter？③ 记录规则和告警规则各解决什么问题？④ 怎么证明「告警规则真的算出来了」？⑤ `for` 设成 `0s` 有什么问题？
>
> 所属：[[Linux/09_日志与监控/00_导读与知识地图|09 日志、监控与可观测性]] 的「一次告警的一生」第 2 站 · 主要练 **S6 规则与告警编写**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住六句话
> - **数据模型是「指标名 + label 集合 → 时间序列」，label 变了就是另一条时序。**
>   - 怎么验证：查一次 `up`，看返回里的 `__name__` + `instance` + `job` 三个 label 才唯一确定一条序列；这也是基数决定成本的原因。
> - **counter 必须用 `rate()`，裸值只是累计秒数。**
>   - 证据：`node_cpu_seconds_total{cpu="0",mode="idle"} 390.42`。
> - **窗口要 ≥ 2×`scrape_interval`，否则 `rate()` 会得到空值或剧烈抖动。**
>   - 证据：本机实验用 `scrape_interval: 2s` 配 `[30s]` 窗口算 CPU 使用率得到 `85.55`；窗口太短就会抖。
> - **规则文件要像代码一样进版本管理并在 CI 里校验。**
>   - 证据：实测 `promtool check rules rules.yml` → `SUCCESS: 2 rules found`。
> - **告警算没算出来，看 `ALERTS` 这条特殊时序，不用等通知渠道。**
>   - 证据：实测 `ALERTS{alertname="LabAlwaysFiring",alertstate="firing",severity="warning"}` 值为 `1`；记录规则 `lab:node_load1` 也返回 `0.13`。
> - **`for: 0s` 会让抖动直接变成通知。**
>   - 怎么验证：本文用 `for: 0s` 只是为了让实验立刻出结果；生产上按告警类型给几分钟，并配合分位数/持续窗口过滤毛刺。

## 1. 数据模型：一条时序由什么确定

```mermaid
flowchart LR
  A["指标名<br/>node_filesystem_avail_bytes"] --> D["时间序列"]
  B["label 集合<br/>device=… fstype=… mountpoint=…"] --> D
  D --> E["样本：时间戳 + 值<br/>1789792700.226 → 1.00738502656e+12"]
```

三条必须记住的推论：

1. **label 变了就是另一条时序**：加一个 `mountpoint`，时序数量就乘以挂载点个数。
2. **同样的指标名可以有完全不同的 label 组合**：写查询时永远先想「我要按什么维度聚合」。
3. **时间戳来自抓取时刻**：所以「短脉冲」可能被两次抓取之间漏掉（子笔记 13）。

## 2. PromQL：四类最常用的写法

| 目的 | 写法 | 注意 |
| --- | --- | --- |
| 速率 | `rate(node_network_receive_bytes_total[5m])` | 窗口 ≥ 2×`scrape_interval`；只能用于 counter |
| 比例/使用率 | `100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100` | 先按 instance 保留维度，再聚合 |
| 聚合与分组 | `sum by (instance) (rate(http_requests_total[5m]))` | `by` 决定输出维度；`without` 相反 |
| 告警表达式 | `up == 0`、`rate(errors[5m]) / rate(total[5m]) > 0.01` | 先算比率再判断，避免绝对值受流量影响 |

实测：解包运行 Prometheus（`scrape_interval: 2s`、`evaluation_interval: 2s`）抓解包运行的 node_exporter，用 HTTP API 查询：

```bash
curl -s "http://127.0.0.1:19090/api/v1/query?query=up"
curl -s "http://127.0.0.1:19090/api/v1/query?query=lab:node_load1"
curl -s "http://127.0.0.1:19090/api/v1/query?query=ALERTS"
curl -s --data-urlencode "query=100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\"}[30s])) * 100)" \
    http://127.0.0.1:19090/api/v1/query
```

```text
[{'metric': {'__name__': 'up', 'instance': '127.0.0.1:19100', 'job': 'node'}, 'value': [1789792700.226, '1']}]
[{'metric': {'__name__': 'lab:node_load1', 'instance': '127.0.0.1:19100', 'job': 'node'}, 'value': [1789792700.24, '0.13']}]
[('LabAlwaysFiring', 'firing', 'warning', '1')]
[1789792700.267, '85.5547740888667']
```

四条查询分别验证了：**目标抓得到（`up=1`）→ 记录规则在算（`lab:node_load1=0.13`）→ 告警进入 firing → 用真实指标算得出使用率（85.6%）**。这套顺序就是「告警链路自检」的最小闭环。

## 3. 记录规则与告警规则

| 类型 | 作用 | 例子 | 什么时候用 |
| --- | --- | --- | --- |
| recording rule | 预计算并保存表达式结果，固化口径、降低查询成本 | `record: job:http_error_ratio:5m` | 复杂表达式被反复查询、或被告警规则/面板共用 |
| alerting rule | 表达式为真时产生告警 | `alert: NodeDown` / `expr: up == 0` | 需要让人做动作的条件 |

本机实测用到的规则文件（一条记录规则 + 一条必然触发的告警规则）：

```yaml
groups:
  - name: lab
    rules:
      - record: lab:node_load1
        expr: node_load1
      - alert: LabAlwaysFiring
        expr: vector(1) > 0
        for: 0s
        labels: {severity: warning}
        annotations: {summary: 实验室告警（用于验证规则链路）}
```

```bash
promtool check rules rules.yml
```

```text
Checking rules.yml
  SUCCESS: 2 rules found
```

三条工程习惯：

1. **规则文件纳入版本管理**，`promtool check rules` 进 CI——语法错误应该在合并前被发现。
2. **告警表达式建立在记录规则或清晰表达式之上**，不要把一个十行表达式直接塞进告警。
3. **`labels` 里必须有 `severity`**，否则后面的路由、抑制、通知分级都无从谈起（子笔记 11、12）。

### 3.1 `ALERTS`：告警状态本身也是时序

任何 alerting rule 触发后，Prometheus 都会生成一条特殊时序：

```text
ALERTS{alertname="…", alertstate="pending|firing", severity="…"} = 0 或 1
```

它带来两个非常实用的能力：

- **验证规则**：`ALERTS{alertname="X"}` 有值就说明规则算出来了，**不需要等通知渠道**。
- **排查「告警不响」**：先看 `ALERTS`，再看 `/api/v1/rules` 的 `state` 与 `lastEvaluation`，最后才查 Alertmanager 与通知渠道（子笔记 17）。

### 3.2 `for`：过滤毛刺的那道闸

```yaml
- alert: HighErrorRatio
  expr: sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.01
  for: 10m
  labels: {severity: critical}
```

`for` 的含义是「表达式持续为真这么久，才从 `pending` 变成 `firing`」。**实测里用 `for: 0s` 是为了让验证立刻成功**；生产上 `for: 0s` 意味着任何一次抖动都会直接通知人。

| `for` 取值 | 效果 | 适用 |
| --- | --- | --- |
| `0s` | 立刻触发 | 只适合实验室验证，或「必须立刻知道」的硬故障（如 `up == 0` 持续 1 分钟） |
| 几分钟 | 过滤瞬时毛刺 | 大多数水位类告警 |
| 十几分钟以上 | 只报持续性问题 | 与 SLO 挂钩的慢速消耗（子笔记 12） |

## 4. 生产动作：把规则写好、验好、管好

> [!example]- 实验 6：写一条记录规则 + 一条告警规则，并用 `ALERTS` 验证
> 目标：在本地跑通「抓取 → 规则评估 → 告警 firing」全链路，不依赖任何外部系统。
> ```bash
> D=$(mktemp -d /tmp/prom-lab.XXXXXX); cd "$D"
> apt-get download prometheus prometheus-node-exporter >/dev/null 2>&1
> mkdir p ne; for f in *.deb; do case "$f" in prometheus-node-exporter*) dpkg-deb -x "$f" ne;; prometheus*) dpkg-deb -x "$f" p;; esac; done
> ne/usr/bin/prometheus-node-exporter --web.listen-address=127.0.0.1:19100 --log.level=error & NE=$!
> printf "%s\n" "global:" "  scrape_interval: 2s" "  evaluation_interval: 2s" \
>   "rule_files: [\"$D/rules.yml\"]" "scrape_configs:" "  - job_name: node" \
>   "    static_configs: [{targets: [\"127.0.0.1:19100\"]}]" > prom.yml
> printf "%s\n" "groups:" "  - name: lab" "    rules:" \
>   "      - record: lab:node_load1" "        expr: node_load1" \
>   "      - alert: LabAlwaysFiring" "        expr: vector(1) > 0" "        for: 0s" \
>   "        labels: {severity: warning}" > rules.yml
> p/usr/bin/promtool check rules rules.yml                       # 验证：SUCCESS: N rules found
> p/usr/bin/prometheus --config.file=prom.yml --storage.tsdb.path=$D/data \
>   --web.listen-address=127.0.0.1:19090 --log.level=error & PR=$!
> sleep 8
> curl -s "http://127.0.0.1:19090/api/v1/query?query=up" | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['result'])"
> curl -s "http://127.0.0.1:19090/api/v1/query?query=lab:node_load1" | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['result'])"
> curl -s "http://127.0.0.1:19090/api/v1/query?query=ALERTS" | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['result'])"
> kill $PR $NE; cd /; rm -rf "$D"
> ```
> **预期**：`up` 的值为 `1`；记录规则返回 `lab:node_load1`；`ALERTS` 里出现 `alertstate="firing"`（本机实测 `[('LabAlwaysFiring', 'firing', 'warning', '1')]`）。
> **风险**：低（解包运行、监听本机高位端口、数据落在临时目录）。
> **回滚**：`kill` 两个进程并删除临时目录。
> **耗时**：20 分钟。
>
> 环境：Ubuntu 24.04.4（WSL2）/ 非 root。**生产环境请用发行版包正常安装**，并给 Prometheus 配好数据目录权限与自启。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 直接对 counter 求平均或比大小 | 累计值只会越来越大，结论无意义；必须 `rate()`/`increase()` |
| `rate(...[1m])` 配 1 分钟抓取间隔 | 窗口里只有一两个样本，结果抖动或为空；窗口应 ≥ 2×`scrape_interval` |
| 告警表达式写绝对值阈值 | 流量变化时误报/漏报；错误率类告警要先算比率 |
| `for: 0s` 图快 | 抖动直接变成通知，长期下来没人再认真看告警 |
| 规则改了不校验就上线 | 语法错误会让整组规则加载失败；用 `promtool check rules` 进 CI |
| 用「通知收到没」判断告警是否工作 | 通知链路上还有路由、分组、抑制、静默；先看 `ALERTS` |
| 把复杂表达式直接写进告警规则 | 可读性与复用性都差；先做 recording rule，再做告警 |

## 决策练习

**场景**：你写了一条告警「磁盘使用率 > 85%」，上线后发现**夜里总有几分钟会触发**，白天不触发，但每次登录查看时磁盘使用率都正常。同一条规则还经常在业务发布后 1 分钟内触发一次。

**A. 把阈值从 85% 调到 95%**
**B. 保留阈值，加 `for: 10m`，并改用「连续窗口内最大值」判定；同时确认是什么在夜里周期性写盘**
**C. 把这条告警关掉，改成每天巡检一次**

**为什么选 B**：现象是「短暂冲高后恢复」，属于**瞬时毛刺**，正确的工程手段是加持续时间要求（`for`）并让表达式对尖峰更稳健（例如用持续窗口而非瞬时值），同时把「谁在夜里写盘」当成一个真实问题去查——它可能正是备份、日志轮转或批处理。

**为什么不选 A**：调阈值等于降低灵敏度——毛刺过滤掉了，真正的「持续 90%」也被放过了，将来真出事时你不会收到告警。

**为什么不选 C**：取消告警是拿掉了防线；磁盘写满的后果（整机故障）与每天巡检一次的时间粒度完全不对等。

## 要点自测

> [!question]- Prometheus 的数据模型是什么？为什么 label 会影响成本？
> - **模型**：指标名 + label 集合唯一确定一条时间序列；一次抓取产生一个样本（时间戳 + 值）。
> - **成本**：label 的每个不同取值组合都是一条独立时序，存储与查询成本随时序数量增长（cardinality）。
> - **推论**：`user_id`、`request_id` 这类高基数信息不能做 label，应进日志或追踪。
> - **第一反应不要是什么**：不要为了「维度更全」往 label 里塞高基数字段。

> [!question]- 怎么在不依赖通知渠道的情况下验证一条告警规则？
> - **第一步 `promtool check rules`**：语法正确（本机实测 `SUCCESS: 2 rules found`）。
> - **第二步看 `ALERTS`**：`ALERTS{alertname="…",alertstate="firing"}` 出现即说明规则触发（本机实测值为 `1`）。
> - **第三步看 `/api/v1/rules`**：确认 `state` 与 `lastEvaluation`，判断是没算出来还是没触发。
> - **之后再查 Alertmanager**：路由、分组、抑制、静默（子笔记 11）。
> - **第一反应不要是什么**：不要一上来就怀疑邮箱/钉钉/webhook。

> [!question]- `for` 参数解决什么问题？该怎么取值？
> - **作用**：表达式持续为真达到 `for` 的时长，告警才从 `pending` 进入 `firing`，用来过滤瞬时毛刺。
> - **取值**：水位类告警常用几分钟；硬故障（`up == 0`）可以短一些；与 SLO 挂钩的慢速消耗可以更长（子笔记 12）。
> - **实测提醒**：本文为了验证链路用了 `for: 0s`，这是**实验手段**，不是生产配置。
> - **第一反应不要是什么**：不要用「调高阈值」代替「加持续时间」。

> 上一篇：[[Linux/09_日志与监控/09_指标的计量_node_exporter与指标语义|09 指标的计量：exporter 与指标语义]] ｜ 下一篇：[[Linux/09_日志与监控/11_告警的降噪_路由分组抑制与探测|11 告警的降噪：路由、分组、抑制与探测]]
