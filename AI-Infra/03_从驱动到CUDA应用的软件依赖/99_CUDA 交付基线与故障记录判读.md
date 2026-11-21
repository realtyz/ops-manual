# CUDA 交付基线与故障记录判读

接手任何预编译 CUDA 应用、PyTorch 训练任务或大模型推理镜像时，最稳妥的工程习惯就是把一次健康运行固化为**交付基线**。交付清单回答“跑的是哪份产物”，宿主机记录回答“本次内核启动管着哪张物理卡”，进程记录回答“它装了什么库、选了哪张卡、跑到了哪一步”。三本账对齐之后，无论后续换版本、升内核还是改启动环境，排查故障就变成了干净的单变量对照实验。

## 参考交付基线

下面的基线记录对应一台运行 Ubuntu 24.04 LTS（x86_64）的裸机服务器，配置单张未切分的 NVIDIA H100（计算能力 9.0），软硬件版本统一对齐全书参考栈（R595 `595.91.07` + CUDA `13.2.1`）：

| 基线维度 | 健康状态记录 |
| --- | --- |
| **构建与交付件** | `/opt/probe/probe-sm90`，采用 CUDA Toolkit `13.2.1` 构建；开启 `--cudart=shared` 与 `-gencode=arch=compute_90,code=sm_90`，仅含 `sm_90` cubin，动态段设置 `DT_RUNPATH=$ORIGIN/lib`。 |
| **宿主机与硬件** | 当前引导内核 `6.8.0-51-generic`，运行中内核模块为 `595.91.07`（R595）。目标 H100 的 BDF 为 `0000:65:00.0`，已记录物理 UUID；`lspci -nnk` 显示 `Kernel driver in use: nvidia`，`nvidia-smi` 查询正常。 |
| **共享库与环境** | `LD_LIBRARY_PATH` 与 `CUDA_VISIBLE_DEVICES` 均未设置；进程实际加载 `/opt/probe/lib/libcudart.so.13` 与宿主机系统目录提供的 `libcuda.so.1`。 |
| **执行与校验结果** | 枚举成功且可见设备数为 1；选中逻辑设备 0，取回的 BDF 与 `0000:65:00.0` 一致；显存分配、`launch` 检查、`synchronize` 与拷回全部通过，最终校验 `output=7`。 |

> [!tip] 跨记录比对的三个对齐锚点
> - **UUID**：跨重启锚定同一张物理 GPU；
> - **BDF**：把进程内的逻辑设备号映射回物理槽位（忽略不同工具打印 PCI 域时的前导零差异，如 `00000000:65:00.0` 与 `0000:65:00.0` 按数值对齐）；
> - **启动周期与时间戳**：用 `journalctl -b` 把内核日志与应用报错严丝合缝地对齐到同一次开机。

## 综合实战检验

### 题目一：三条路径与可见性隔离

基于上面的参考交付基线，先在脑中（或纸上）画出构建侧、计算支路与管理支路三条路径，标清 `nvcc`、`libcudart`、`libcuda.so.1`、NVML、Linux 内核驱动与 GSP 固件各自的位置。

随后考虑两个独立变动：① 生产节点上找不到 `nvcc` 命令；② 启动探针时单独传入 `CUDA_VISIBLE_DEVICES=`。请判断哪条路径最先出现变化，以及哪些观测结果依然成立？

### 题目二：五类典型故障记录判读

下表列出五起彼此独立的故障现场。案例 A、C、D、E 均沿用参考交付基线中未被改动的所有条件；案例 B 取自 Canonical Ubuntu Server 官方文档的真实失配日志。请针对每一起案例，分别写出：
1. **最早有证据支持的失败点**；
2. **一项最能缩小范围的下一步检查**；
3. **证明原问题已闭环解决的复核动作**。

| 案例 | 现场收集到的故障材料 |
| --- | --- |
| **A｜内核升级重启后脱管** | 主机升级内核并重启进入 `6.8.0-52-generic`。`lspci` 仍列出 `0000:65:00.0`，但不再显示 `Kernel driver in use: nvidia`；`lsmod` 中无 `nvidia`，`modinfo -k "$(uname -r)" nvidia` 报告找不到模块。 |
| **B｜NVML 初始化版本失配** | 执行 `nvidia-smi` 报 `Driver/library version mismatch`。同期内核日志记录客户端版本为 `570.172.08`，而运行中内核模块版本为 `570.158.01`（当前缺少进程加载 NVML 的路径与磁盘模块版本）。 |
| **C｜首次启动 Kernel 报 209** | 进程顺利通过枚举、选卡与 `cudaMalloc`，但在 `launch` 检查点返回 `cudaErrorNoKernelImageForDevice`（209）。核查提交该 Kernel 的产物发现，其 fatbin 仅含 `sm_80` cubin，PTX 列表为空。 |
| **D｜改环境变量致启动前崩溃** | 二进制仍为 `/opt/probe/probe-sm90`（依赖 `libcudart.so.13`，`DT_RUNPATH=$ORIGIN/lib`）。`/opt/probe/lib` 与 `/opt/old-app/lib` 下均有同名库；启动时新增 `LD_LIBRARY_PATH=/opt/old-app/lib`，进程尚未进入 `main` 即报共享库符号错误。 |
| **E｜降级至 R580 后的分叉表现** | 同台 H100 将驱动切换为 R580 分支的 `580.65.06`，实际加载 CUDA `13.2.1` 共享 Runtime 与系统驱动库。运行 `probe-sm90` 正常输出 `output=7`；换用仅含 `13.2.1` 生成的 `compute_80` PTX 的 `probe-ptx80` 后，枚举与选卡通过，首次调用 Kernel 时返回 `cudaErrorUnsupportedPtxVersion`（222）。 |

## 分步参考推理

### 题目一参考推理

- **路径分工**：构建机借助 CUDA Toolkit 中的 `nvcc` 编译主机与设备代码生成交付件；应用进程在计算支路经 `libcudart` → `libcuda.so.1` → 设备节点进入内核驱动；`nvidia-smi` 在管理支路经 NVML 接入同一内核底座；GSP 固件由驱动分发并运行在 GPU 协处理器上，负责初始化与管理。
- **缺 `nvcc` 的影响**：仅意味着当前 Shell 无法直接执行源码编译，对已经构建好的预编译交付件 `probe-sm90` 运行毫无影响。
- **置空 `CUDA_VISIBLE_DEVICES=` 的影响**：仅作用于当前应用进程的计算支路，最早在 `enumerate` 阶段分叉（可见列表为空，进程止步于选卡之前）；而宿主机上另行执行的 `nvidia-smi` 走 NVML 管理支路，依然能正常列出 H100。

### 题目二参考推理

**案例 A｜当前启动内核缺少模块**
- **失败定位**：PCIe 枚举已成功，故障收敛到新内核 `6.8.0-52-generic` 的 NVIDIA 内核模块缺失与设备未绑定。
- **下一步取证**：按节点所用的模块交付方式，核查 `6.8.0-52-generic` 对应的预编译包（`linux-modules-nvidia-...`）是否安装，或查看 `dkms status` 与该内核的 DKMS 构建日志；若补装后磁盘模块存在但仍未加载，再查本次启动的 `journalctl -k -b` 排查符号版本或模块签名拒绝。
- **复核标准**：确认 `uname -r`、运行中模块版本（`/proc/driver/nvidia/version`）与 `0000:65:00.0` 的 `Kernel driver in use: nvidia` 全部对齐后，先后跑通 `nvidia-smi` 与 `probe-sm90` 并校验 `output=7`。

**案例 B｜运行中内核模块与 NVML 库版本失配**
- **失败定位**：NVML 管理支路在初始化交互阶段失败，用户态库（`570.172.08`）与运行中内核模块（`570.158.01`）版本不一致。
- **下一步取证**：在同一报错环境下运行 `LD_DEBUG=libs,files nvidia-smi` 确认实际加载的 `libnvidia-ml.so.1` 路径及归属包，同时用 `modinfo -k "$(uname -r)" nvidia` 查看磁盘模块版本。
  - 若磁盘与系统 NVML 均为 `570.172.08` 而内存仍是 `570.158.01`，属于升级后未重载内核模块；若磁盘与运行中模块一致而进程加载了额外目录，则属库搜索路径冲突。
- **复核标准**：针对确认的根因完成重载或修正搜索路径后，回到原启动环境重新执行 `nvidia-smi` 确认管理支路恢复；若需确认计算可用，再执行业务探针核验同步与输出。

**案例 C｜产物缺少适用于当前 GPU 的设备代码**
- **失败定位**：库装载、枚举、选卡与显存分配全部通过，故障发生在首次调用 Kernel 的设备代码装载环节。H100 为计算能力 9.0，`sm_80` 普通 cubin 无法跨主版本装入，产物又未内嵌可供 JIT 的 PTX。
- **下一步取证**：确认 `cuobjdump` 检查的正是实际提交该 Kernel 的文件（若来自框架或自定义算子，须对准对应 `.so` 核查）。
- **复核标准**：固定驱动、Runtime 与环境变量，仅替换为包含 `sm_90` cubin（或合法 PTX）的同源产物重新运行；若 `launch`、`synchronize` 与 `output=7` 全部通过，即完成单变量闭环验证。

**案例 D｜环境变量劫持导致动态链接失败**
- **失败定位**：程序尚未进入 `main`，失败落在主机侧 glibc 动态链接装载阶段。按默认搜索次序，新增的 `LD_LIBRARY_PATH` 优先级高于 `DT_RUNPATH`，导致链接器抢先命中 `/opt/old-app/lib/libcudart.so.13` 并触发符号不匹配。
- **下一步取证**：记录报错中的具体缺失符号及版本，并在相同环境下运行 `LD_DEBUG=libs,files /opt/probe/probe-sm90`，核对报错共享对象的真实加载路径。
- **复核标准**：仅移除新增的 `LD_LIBRARY_PATH` 做单变量对照，确认动态链接器重新载入 `/opt/probe/lib/libcudart.so.13`、进程进入 `main` 并顺利输出 `output=7`。

**案例 E｜次版本兼容下限与 PTX JIT 边界**
- **失败定位**：R580（`580.65.06`）跨过了 CUDA 13.x 次版本兼容下限，自带原生 `sm_90` cubin 且不调新驱动 API 的 `probe-sm90` 可直接跑通；但换用仅含 CUDA `13.2.1` PTX（PTX ISA 9.2）的 `probe-ptx80` 时，R580 旧版 JIT 无法解析新版 PTX，首次执行即抛出 222。
- **下一步取证**：用 `cuobjdump --dump-ptx probe-ptx80` 核对 PTX 头的 `.version` 字段，并确认进程实际加载的驱动 JIT 版本。
- **复核标准**：保持 `probe-ptx80` 与共享 Runtime 不变，将宿主机驱动切回满足 CUDA 13.2 常规配对基线（`≥ 595.45.04`）的 R595（如 `595.91.07`）。
  - 重跑并验证 `launch`、同步与回读 `7` 全部通过，从而将“次版本兼容下限”“架构匹配”和“PTX JIT 支持”三个独立条件清晰分开。

## 本章知识串联与后续衔接

> [!summary] 全章骨架：把三本账扣在一起
> 一次成功的 CUDA 计算，本质上是**交付件**、**应用进程**与**宿主机底座**三层严丝合缝的接力：
> 1. **交付件**声明主机依赖（`DT_NEEDED` / `DT_RUNPATH`）并携带适配目标 GPU 的设备代码（cubin / PTX）；
> 2. **应用进程**经 glibc 正确解析共享库，通过可见性过滤枚举设备并建立 Primary Context，在首次调用 Kernel 时按需装载代码并异步同步回收结果；
> 3. **宿主机底座**确保当前引导内核已加载匹配版本的 NVIDIA 内核模块、绑定目标 PCIe BDF、拉起 GSP 固件并开放设备节点访问权限。

理解了单机上的软件依赖后，接下来两条主线自然展开：
- [第 04 章 单机多卡与 GPU 数据路径](../04_单机多卡与GPU数据路径/00_内容大纲.md) 将沿着这里的 `cudaMemcpy` 继续往下挖，追踪主机内存、单卡显存与多张 GPU 之间的数据究竟走过哪些 PCIe Switch 与 NVLink/NVSwitch 物理链路。
- [第 10 章 GPU 如何进入容器](../10_GPU如何进入容器/00_内容大纲.md) 会把本章的三层依赖放进容器隔离边界：你会立刻看懂为什么 PyTorch 训练与大模型推理容器镜像自带 `libcudart` 与算子库，而设备节点与 `libcuda.so.1` 必须由宿主机运行时精准注入。

> [!info]- 参考资料
> - **CUDA 平台、驱动基线与计算能力**：[CUDA 13.2 GA Release Notes · CUDA Driver](https://docs.nvidia.com/cuda/archive/13.2.0/cuda-toolkit-release-notes/index.html#cuda-driver)、[NVIDIA GPU 计算能力表](https://developer.nvidia.com/cuda/gpus)、[NVIDIA Hopper 兼容指南](https://docs.nvidia.com/cuda/archive/12.6.0/hopper-compatibility-guide/index.html)
> - **驱动组件、GSP 与管理接口**：[NVIDIA Linux 驱动组件](https://download.nvidia.com/XFree86/Linux-x86_64/580.65.06/README/installedcomponents.html)、[NVIDIA GSP 固件说明](https://download.nvidia.com/XFree86/Linux-x86_64/580.65.06/README/gsp.html)、[NVIDIA SMI 手册](https://docs.nvidia.com/deploy/nvidia-smi/index.html)
> - **设备管理、错误码、可见性与兼容性**：[CUDA 13.2.1 Runtime API · 设备管理](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)、[CUDA 13.2.1 Runtime API · 错误码](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__TYPES.html#group__CUDART__TYPES)、[CUDA 13.2.1 CUDA_VISIBLE_DEVICES](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html#cuda-visible-devices)、[CUDA 次版本兼容 · PTX 与应用限制](https://docs.nvidia.com/deploy/cuda-compatibility/latest/minor-version-compatibility.html#application-considerations-for-minor-version-compatibility)
> - **Linux 模块与动态链接排障**：[Ubuntu Server · 驱动模块安装](https://ubuntu.com/server/docs/how-to/graphics/install-nvidia-drivers/#installing-the-kernel-modules)、[Ubuntu Server · 版本失配示例](https://ubuntu.com/server/docs/how-to/graphics/install-nvidia-drivers/#driver-library-version-mismatch-error)、[Linux 模块签名文档](https://docs.kernel.org/admin-guide/module-signing.html)、[glibc ld.so(8) 手册](https://man7.org/linux/man-pages/man8/ld-linux.8.html)
