# Linux NVIDIA 模块加载与进程库解析

昨晚刚用 `apt upgrade` 给服务器打了内核安全补丁或升级了 NVIDIA 驱动，今天一早机器还没重启（或者刚重启完），`nvidia-smi` 就直接甩出一句 `Failed to initialize NVML: Driver/library version mismatch`，甚至干脆整机“丢卡”。更让人头大的是：同一个二进制程序，换个终端窗口跑，竟一边正常、一边报符号找不到。

这些看似玄学的故障，其实全被 Linux 最基础的两套机制管着：**内核态的模块加载与设备绑定**，以及 **用户态 glibc 动态链接器的共享库解析**。只要把“磁盘上有什么”“内核里跑着什么”“进程里载了什么”这三层账分开对，故障立刻水落石出。

## 启动内核、磁盘模块与 PCIe 设备绑定

软件包更新改写的是**磁盘上的文件**，而当前真正运行的 Linux 内核版本由本次引导决定（用 `uname -r` 查看）。外部内核模块强依赖对应内核的接口；如果服务器升级并切进了新内核，但新内核对应的模块目录里还没有匹配的 NVIDIA 模块，PCIe 总线虽然照样能枚举出 H100，驱动却无法接管这张卡。

在 Ubuntu 服务器上，与当前内核匹配的 NVIDIA 模块通常由以下两种方式之一交付：

- **预编译模块包**：发行版直接提供绑定特定内核 ABI（应用二进制接口）与内核风味的二进制包，例如 `linux-modules-nvidia-<分支>-server-<内核版本>`（无桌面计算节点常见 `-server` 变体）。一旦内核小版本变更而未同步装入对应预编译包，新内核启动后就会缺失模块。
- **DKMS（Dynamic Kernel Module Support，动态内核模块支持）**：在目标机器上借助对应内核的头文件与构建环境现场编译安装模块。旧内核构建成功并不代表新内核自动就绪，内核升级后必须核对新内核下的 DKMS 构建状态。

即便磁盘模块树里找到了 `nvidia.ko`，Linux 内核在真正接纳它之前还要过两道安检：**模块符号版本校验（Module Versioning）** 确保模块引用的内核符号与当前内核 ABI 一致；若主机开启强制**模块签名验证**（如 Secure Boot 场景），内核还要求模块具备受信任公钥签发的有效签名。

> [!tip] 工程直觉：不要看到签名告警就草率定案
> 在允许装载未签名模块的内核配置下，内核日志里出现“模块未签名（taint）”告警可以与模块成功加载同时存在。只有把 `modinfo` 字段、本次启动日志、`lsmod` 装入状态与 `lspci` 绑定结果合起来看，才能精准区分“磁盘缺文件”“模块被内核拒绝”与“模块已加载但未绑定设备”。

按下面这四条命令顺次取证，几秒钟就能把当前启动的内核与设备绑定状态摸清：

```bash
# 1. 确认本次启动运行的内核版本
$ uname -r
6.8.0-51-generic

# 2. 检查当前运行内核的模块树中是否存在匹配文件，重点看 path / version / vermagic / signer
$ modinfo -k "$(uname -r)" nvidia | grep -E "^(filename|version|vermagic|signer):"
filename:       /lib/modules/6.8.0-51-generic/updates/dkms/nvidia.ko.zst
version:        595.91.07
vermagic:       6.8.0-51-generic SMP preempt mod_unload modversions
signer:         Ubuntu Secure Boot Module Signature key

# 3. 确认内核内存中“实际正在运行”的模块版本（注意与磁盘 modinfo 对照）
$ cat /proc/driver/nvidia/version
NVRM version: NVIDIA UNIX Open Kernel Module for x86_64  595.91.07  Release Build

# 4. 确认目标 BDF 已由 nvidia 驱动接管，并用 journalctl -k -b 仅看本次启动内核日志
$ lspci -nnk -s 0000:65:00.0
65:00.0 3D controller [0302]: NVIDIA Corporation GH100 [H100 SXM5 80GB] [10de:2330] (rev a1)
	Kernel driver in use: nvidia
```

把这四个关键检查点收束成一张状态核对表：

| 核对环节 | 核心取证命令 | 判断要点 |
| --- | --- | --- |
| **当前运行内核** | `uname -r`、`journalctl -k -b` | 锁定本次引导的内核版本，且只看本次启动的内核日志，避免被旧日志误导。 |
| **磁盘模块准备** | `modinfo -k "$(uname -r)" nvidia`、`dkms status` | 确认 `filename`、`version`、`vermagic` 与 `signer`；DKMS 方案另查对应内核构建日志。 |
| **内存运行中模块** | `lsmod`、`/proc/driver/nvidia/version` | 确认模块已经装入内核，并读出内存中正在运行的精确版本号。 |
| **PCIe 设备绑定** | `lspci -nnk -s 0000:65:00.0` | 目标 GPU 条目明确出现 `Kernel driver in use: nvidia`，才算完成设备接管。 |

## glibc 共享库搜索顺序：为什么换个环境就加载错库

当宿主机内核模块与设备绑定都稳稳当当时，同一个预编译二进制依然可能因为启动环境里多了一条环境变量，就悄悄载入完全不同的 `libcudart` 或驱动库。

以打包在 `/opt/probe/probe-sm90` 的探针为例：它的直接依赖为 `DT_NEEDED=libcudart.so.13`，同时构建者在 ELF 头里写入了 `DT_RUNPATH=$ORIGIN/lib`（其中 `$ORIGIN` 会由链接器自动展开为可执行文件所在的 `/opt/probe` 目录，指向 `/opt/probe/lib`）。

对于这类普通 glibc 进程（直接依赖名不含斜杠、未使用旧式 `DT_RPATH`、未处于安全执行模式），动态链接器按固定优先级寻找同名共享库：**先查 `LD_LIBRARY_PATH` → 再查该文件的 `DT_RUNPATH` → 最后查 `/etc/ld.so.cache` 与系统默认目录**。

当 `/opt/probe/lib` 与 `/opt/old-app/lib` 里都存在 `libcudart.so.13` 时，不同启动环境的命中结果截然不同：

| 启动环境 | 动态链接器最先命中的共享库 |
| --- | --- |
| 未设置 `LD_LIBRARY_PATH` | `/opt/probe/lib/libcudart.so.13`（命中 `DT_RUNPATH`） |
| 设置 `LD_LIBRARY_PATH=/opt/old-app/lib` | `/opt/old-app/lib/libcudart.so.13`（环境变量抢在 `DT_RUNPATH` 之前） |

排查库解析冲突时，不要靠肉眼猜路径——先用 `readelf -d` 看 ELF 静态声明，再用 `LD_DEBUG=libs,files` 让动态链接器亲口汇报实际搜过哪些目录、最终载入了哪份文件：

```bash
# 1. 查看 ELF 声明的直接依赖（NEEDED）与内嵌查找路径（RUNPATH / RPATH）
$ readelf -d /opt/probe/probe-sm90 | grep -E "NEEDED|RUNPATH|RPATH"
 0x0000000000000001 (NEEDED)             Shared library: [libcudart.so.13]
 0x000000000000001d (RUNPATH)            Library runpath: [$ORIGIN/lib]

# 2. 对比两种启动环境下的真实动态链接过程（节选关键行）
$ env -u LD_LIBRARY_PATH LD_DEBUG=libs,files /opt/probe/probe-sm90 2>&1 | grep -E "libcudart|calling init"
     18421:	find library=libcudart.so.13 [0]; searching
     18421:	 search path=/opt/probe/lib		(RUNPATH from file /opt/probe/probe-sm90)
     18421:	  trying file=/opt/probe/lib/libcudart.so.13
     18421:	calling init: /opt/probe/lib/libcudart.so.13     # 确认已真正映射并初始化

$ env LD_LIBRARY_PATH=/opt/old-app/lib LD_DEBUG=libs,files /opt/probe/probe-sm90 2>&1 | grep -E "libcudart|calling init"
     18509:	find library=libcudart.so.13 [0]; searching
     18509:	 search path=/opt/old-app/lib		(LD_LIBRARY_PATH)
     18509:	  trying file=/opt/old-app/lib/libcudart.so.13   # 抢先命中旧目录
```

> [!warning] 常见误区：读懂 `LD_DEBUG` 与 `DT_RUNPATH` 的三个边界
> 1. **`trying file=` 不等于加载成功**：它只代表链接器试探了该路径；必须看到后续生成映射或 `calling init:` 记录，才能确认文件已被实际载入。
> 2. **`DT_RUNPATH` 只管直接依赖**：主程序的 `DT_RUNPATH` 不会自动传递给子依赖；如果某条传递依赖找错库，必须用 `readelf -d` 查声明它的那个 `.so`。
> 3. **特例会改写搜索次序**：依赖名自带路径、旧式 `DT_RPATH`（优先级高于 `LD_LIBRARY_PATH`）以及 setuid 等安全执行模式（会忽略 `LD_LIBRARY_PATH`），都会改变上述顺序。若库名找不到，进程在启动前即报无法打开共享对象；若选中的同名库缺少所需符号或符号版本，则可能在装载期或首次调用该符号时报错。

## 同样的 NVML 版本失配，两处截然不同的根因

用户态共享库顺利加载后，还必须能和当前内核里正在运行的驱动模块对齐协议版本。Canonical 官方 Ubuntu Server 驱动文档中记录了一个在生产环境几乎人人遇过的经典报错：

```text
# 终端执行 nvidia-smi 的报错
Failed to initialize NVML: Driver/library version mismatch

# 同一时间内核日志（dmesg / journalctl -k -b）给出的精确两端版本
NVRM: API mismatch: the client has the version 570.172.08, but
NVRM: this kernel module has the version 570.158.01.  Please
NVRM: make sure that this kernel module and all NVIDIA driver
NVRM: components have the same version.
```

多数人看到这条报错，第一反应就是“赶紧重启机器”。重启动作在升级未重载内核模块时确实有效，但**如果你只看到两端版本号不同，其实并不足以断定一定是“没重启”**！

把**磁盘模块版本**、**运行中内核模块版本**和**进程实际加载的 `libnvidia-ml.so.1` 来源**三者摆到一起，同一条失配报错立刻分出两种截然不同的根因：

- **根因一（左列 B / A / B：升级后旧模块仍在运行）**：系统升级把磁盘上的 `nvidia.ko` 和系统库目录里的 NVML 都更新成了新版 `B`（如 `570.172.08`），但内核内存里还没卸载重载，跑的仍是旧版 `A`（如 `570.158.01`）。新启动的 `nvidia-smi` 加载了新版库去敲旧版内核的门，立刻触发失配。
- **根因二（右列 B / B / A：进程从额外目录加载了旧库）**：磁盘模块和内核运行中模块早就一致升级成了新版 `B`，但当前 Shell 或容器通过 `LD_LIBRARY_PATH` 等配置，把 `nvidia-smi` 引去了某个遗留目录，加载了旧版 NVML `A`。此时哪怕重启十次服务器，只要环境变量不改，报错依然纹丝不动！

![两种 NVML 版本失配。磁盘模块、运行中模块、进程加载的 NVML 在左列依次为 B、A、B，右列依次为 B、B、A。两列的红色括线都连接运行中模块与进程加载的 NVML。](images/fig-03-04-01-nvml-module-mismatch.png)

*图 03-04-01：同一 NVML 版本失配报错的两种根因对比（A 表示旧版，B 表示新版）。左列为驱动更新后旧内核模块仍在运行，右列为内核已是新版但进程从额外目录加载了旧库。*

因此，现场遇到版本失配时，标准动作是同步采集三条证据：

- **查内存运行中模块**：读取 `/proc/driver/nvidia/version` 并对照 `journalctl -k -b`；
- **查磁盘模块文件**：执行 `modinfo -k "$(uname -r)" nvidia` 核对磁盘上的版本；
- **查进程实际加载的库**：在**报错的同一个启动环境**下跑 `LD_DEBUG=libs,files nvidia-smi`，查出命中的 `libnvidia-ml.so.1` 路径及所属包。

> [!summary] 核心结论：修复后的双支路复核
> 解决模块重载或库路径问题后，务必回到原报错环境先复测管理命令（验证 NVML 管理支路恢复），再运行实际 CUDA 交付件完成设备枚举、Kernel 同步与结果校验（验证计算支路恢复）。两条支路都通过，才算真正闭环。

现在，构建交付、进程启动、版本兼容、内核模块与动态库解析五块拼图已全部集齐。在收官篇中，我们将把这些散落的取证方法组装成一份完整的交付基线，用五道真实感故障记录检验你的分层定位能力。

> [!info]- 参考资料
> - **Ubuntu NVIDIA 驱动与排障文档**：[Ubuntu Server · 安装 NVIDIA 驱动总览](https://ubuntu.com/server/docs/how-to/graphics/install-nvidia-drivers/)、[预编译模块与 DKMS 安装](https://ubuntu.com/server/docs/how-to/graphics/install-nvidia-drivers/#installing-the-kernel-modules)、[Driver/library version mismatch 示例](https://ubuntu.com/server/docs/how-to/graphics/install-nvidia-drivers/#driver-library-version-mismatch-error)、[NVIDIA 驱动故障排查指南](https://ubuntu.com/server/docs/how-to/graphics/install-nvidia-drivers/#troubleshooting)
> - **Linux 内核模块、ABI 与签名**：[Linux kbuild · 外部内核模块](https://docs.kernel.org/kbuild/modules.html)、[Linux kbuild · Module Versioning](https://docs.kernel.org/kbuild/modules.html#module-versioning)、[Linux 内核模块签名文档](https://docs.kernel.org/admin-guide/module-signing.html)
> - **系统工具与动态链接器**：[modinfo(8) 手册](https://man7.org/linux/man-pages/man8/modinfo.8.html)、[lspci(8) 手册](https://man7.org/linux/man-pages/man8/lspci.8.html)、[glibc ld.so(8) 手册](https://man7.org/linux/man-pages/man8/ld-linux.8.html)、[dlopen(3) 手册](https://man7.org/linux/man-pages/man3/dlopen.3.html)、[NVIDIA NGC 镜像 CUDA 13.2 Runtime 库名参考](https://catalog.ngc.nvidia.com/orgs/nvidia/-/containers/nvhpc/26.5-runtime-cuda13.2-ubuntu22.04/layers)
