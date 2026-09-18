---
tags:
  - Linux
  - SELinux
  - AppArmor
  - MAC
  - 安全加固
created: 2026-09-18
---

# 第 9 站：强制访问控制（SELinux 与 AppArmor）

> [!cite] 参考资料
> `man 8 getenforce`、`man 8 setenforce`、`man 8 sestatus`、`man 8 restorecon`、`man 8 semanage`、`man 8 setsebool`、`man 8 ausearch`、`man 8 audit2allow`、`man 8 aa-status`、`man 8 aa-enforce`、`man 5 apparmor.d`。
>
> 本篇结论来自上述资料；SELinux 与 AppArmor 的实际拒绝行为在 Ubuntu 24.04 WSL2 环境未实测，正文标注「未实测」或按官方文档撰写。

> **这篇讲什么**：十站的第八站——DAC 之外的强制访问控制。SELinux 看类型上下文，AppArmor 看程序路径 profile。重点是 `enforcing` 下服务被挡时，怎么按「找 AVC → 修上下文/端口/布尔值 → 必要时补策略 → 切回 enforcing」四步处置，而不是 `setenforce 0`。
>
> **必须先读什么**：[[Linux/07_权限与安全加固/01_前置_安全坐标系与四层防线|01 前置：安全坐标系与四层防线]]、[[Linux/07_权限与安全加固/03_前置_权限观测工具与证据命令|03 前置：权限观测工具与证据命令]]。
>
> **读完能回答**：① SELinux 三种模式是什么？② AVC 拒绝记录长什么样？③ 为什么 `setenforce 0` 不算解决问题？
>
> 所属：[[Linux/07_权限与安全加固/00_导读与知识地图|07 权限、账号与安全加固]] 的十站主线第 8 站 · 主要练 **S6 强制访问控制排障**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **MAC 是「即使 DAC 同意，策略也不允许」**：SELinux 看安全上下文，AppArmor 看程序路径 profile。
> - **SELinux 三种模式**：`enforcing` 阻断并审计，`permissive` 只审计不阻断，`disabled` 关闭。
> - **排障入口是 AVC 日志**：`ausearch -m AVC,USER_AVC -ts recent` 或 `journalctl -g avc`。
> - **四步处置**：找 AVC → `restorecon`/`semanage`/`setsebool` → 确需才 `audit2allow` 并人工复核 → 切回 `enforcing` 复验。
> - **`setenforce 0` 只是把证据关掉**，不是修复；更常见的坑是切了 permissive 忘记切回。

## 1. SELinux 与 AppArmor 的模型差异

| | SELinux | AppArmor |
| --- | --- | --- |
| 常见发行版 | RHEL/Fedora | Debian/Ubuntu |
| 判定依据 | 文件的 `user:role:type:level` 上下文 | 程序路径对应的 profile |
| 日志 | `avc: denied` | `apparmor="DENIED"` |
| 主要工具 | `ls -Z`、`restorecon`、`semanage`、`setsebool` | `aa-status`、`aa-enforce`、`aa-complain` |

## 2. SELinux 模式与上下文

```bash
getenforce                      # 验证：当前模式 Enforcing/Permissive/Disabled
sestatus                        # 验证：模式、策略版本、挂载状态
ls -Z /var/www/html             # 验证：文件上下文；无 SELinux 时可能输出 ?
stat -c %C /var/www/html        # 验证：上下文的 stat 视角
```

上下文是 `user:role:type:level`，核心判定看 `type`。它随 inode 走，不随路径走，所以移动文件后要用 `restorecon` 恢复标签。

## 3. 四步处置

1. **找拒绝**：

```bash
ausearch -m AVC,USER_AVC -ts recent
journalctl -k -g avc
```

典型记录：

```text
type=AVC msg=audit(...): avc:  denied  { write } for  pid=... comm="httpd" name="uploads" ...
```

2. **判断是标签、端口还是布尔值**：能用 `restorecon` 修标签、`semanage port` 修端口、`setsebool -P` 修布尔值，就不要写新策略。

```bash
restorecon -Rv /var/www/html
semanage port -a -t http_port_t -p tcp 8080
setsebool -P httpd_can_network_connect on
getsebool -a | grep httpd
```

3. **确需补策略才生成并人工复核**：`audit2allow -M <name> -i /var/log/audit/audit.log` 后 `semodule -i <name>.pp`。生成结果要看范围，不能照单全收。

4. **切回 enforcing 复验**：`setenforce 1` 后重放业务操作。

> [!warning] `setenforce 0` 只是把强制访问控制关成只审计
> 拒绝不再被阻止，攻击面回到 DAC；而且很多人切完忘记切回，问题被延后放大。**permissive 只能用于定位，不能作为长期状态。**

## 4. AppArmor 的入口

```bash
aa-status                      # 验证：已加载 profile 与 enforce/complain 模式
aa-enforce /path/to/profile    # 验证：把 profile 切到 enforce
journalctl -k -g apparmor      # 验证：AppArmor DENIED 日志
```

AppArmor 按程序路径匹配 profile；`complain` 模式只记录不阻断，`enforce` 阻断并记录。

## 5. 实验：在支持 MAC 的机器上补做

> [!example]- 实验：SELinux 拒绝与恢复（RHEL 9 或支持 SELinux 的虚拟机）
> **怎么做**：把服务数据目录移到非标准路径，制造 AVC 拒绝；走 `ausearch → restorecon/semanage/setsebool`；若确需自定义策略，用 `audit2allow` 生成后人工复核。
> **预期**：定位到 `avc: denied`，修复后业务恢复，并切回 `enforcing`。
> **风险**：涉及强制策略，先快照并保留控制台。
> **耗时**：约 30 分钟。
> **回滚**：快照回滚；或反向撤销 semanage/setsebool/module。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 用 `setenforce 0` 当修复 | 强制访问控制被关闭，攻击面回到 DAC |
| 切 permissive 后忘记切回 | 拒绝只记录不阻断，问题延后爆发 |
| 直接 `audit2allow` 照单全收 | 可能生成过宽策略，把正常强制边界挖洞 |
| 移动文件后不 `restorecon` | 上下文错位导致拒绝 |
| 在 Ubuntu 上找 SELinux | Ubuntu 默认用 AppArmor，入口和日志不同 |

## 决策练习

> [!question]- 场景：RHEL 上服务在 `enforcing` 下起不来，同事说「先 `setenforce 0` 恢复业务」。你怎么办？
> A. 同意并切 permissive
> B. 先查 AVC 拒绝、保留证据，尝试 `restorecon`/`semanage`/`setsebool` 最小修复，再切回 `enforcing` 复验；permissive 只用于定位
> C. 重装服务，避开策略
>
> **答案：B。**
> A 恢复了业务，但关了强制边界，且可能忘记切回。
> C 是掩盖，不解决策略或上下文问题。
> B 是正解：**MAC 拒绝要用证据修复，不是关掉。**

## 要点自测

> [!question]- SELinux 三种模式是什么？
> - `enforcing`：阻断并审计；`permissive`：只审计不阻断；`disabled`：关闭。
> - **第一反应不要是什么**：不要把 permissive 当长期状态。

> [!question]- 为什么 `setenforce 0` 不算解决问题？
> - 它把强制策略关成只审计，拒绝行为不再阻止，攻击面回到 DAC。
> - 正确做法是按 AVC 修标签/端口/布尔值或补策略，再切回 enforcing。
> - **第一反应不要是什么**：不要只求业务先起来，就把强制边界关掉。

> [!question]- SELinux 与 AppArmor 的判定依据差在哪？
> - SELinux 看文件 inode 的安全上下文；AppArmor 看程序路径对应的 profile。
> - Ubuntu 默认 AppArmor，RHEL 默认 SELinux。
> - **第一反应不要是什么**：不要把两者的工具和日志混用。

> 上一篇：[[Linux/07_权限与安全加固/11_第8站_SSH加固|11 第 8 站：SSH 加固]] ｜ 下一篇：[[Linux/07_权限与安全加固/13_第10站_挂载选项与内核安全参数|13 第 10 站：挂载选项与内核安全参数]]
