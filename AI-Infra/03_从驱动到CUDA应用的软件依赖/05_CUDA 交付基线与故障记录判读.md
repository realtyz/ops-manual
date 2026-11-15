
接手预编译 CUDA 应用时，可以把一次成功运行记录为基线。交付清单回答运行的是哪个文件，宿主机记录回答哪次启动管理着哪张卡，进程记录回答它装了什么库、选了什么设备、执行到了哪一步。三者对齐后，替换产物、重启内核或改变启动环境造成的影响才有可比较的证据。

## H100 上的一次交付基线

以下是 **教学基线**。环境为 Ubuntu 24.04 LTS、x86_64 Linux 裸机、单张未切分 H100。`boot-A0` 和 `GPU-H100-A` 是本例用于关联记录的简写。产物、驱动与架构按官方资料设定，表内调用结果为推演。 [CUDA 13.2 GA · 对应驱动](https://docs.nvidia.com/cuda/archive/13.2.0/cuda-toolkit-release-notes/index.html#cuda-driver) · [NVIDIA GPU 计算能力表](https://developer.nvidia.com/cuda/gpus)

| 材料 | 教学记录 |
| --- | --- |
| 构建与交付 | `probe-sm90` 由 Toolkit 13.2 GA 构建。使用 `--cudart=shared` 和 `-gencode=arch=compute_90,code=sm_90`，`write_value` 的设备代码是 `sm_90` cubin。 |
| 主机与设备 | 启动 `boot-A0`，内核 `6.8.0-51-generic`，运行中内核模块 `595.45.04`。H100 的 UUID 简写为 `GPU-H100-A`，BDF 为 `0000:65:00.0`，由 `nvidia` 绑定，`nvidia-smi` 成功。 |
| 库与环境 | 沿用第 04 篇的教学打包。应用装入 `/opt/probe/lib/libcudart.so.13` 和系统驱动提供的 `libcuda.so.1`，`LD_LIBRARY_PATH`、`CUDA_VISIBLE_DEVICES` 均未设置。 |
| 应用结果 | 枚举成功且设备数为 1。选中进程内设备 0，返回的 BDF 为 `0000:65:00.0`。显存分配、启动检查、同步、拷回均成功，`output=7`。 |

比对时，用 UUID 关联同一张卡，用 BDF 把这一轮的应用设备 0 对上管理侧记录。不同工具可能用不同数量的前导零打印 PCI 域，比较时按地址字段的数值对齐。最后用启动标识或时间把内核日志与应用运行对上。 [NVIDIA SMI · 设备标识](https://docs.nvidia.com/deploy/nvidia-smi/index.html) · [CUDA 13.2.1 Runtime API · 设备管理](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)

## CUDA 交付与故障定位综合检验

先用基线画出构建、计算和管理三条路径，标出 `nvcc`、`libcudart`、`libcuda`、NVML、内核驱动和 GSP 的位置。随后假设运行主机找不到 `nvcc`，另一次启动仅为探针设置 `CUDA_VISIBLE_DEVICES=`。推导哪条路径的最早变化可见，哪些观测还能成立。

下表中的案例各自独立。A、C、D、E 沿用基线未被修改的条件。B 使用 Canonical 文档的独立示例。对每一行写出最早有证据支持的失败点、一项能缩小范围的检查，以及哪条复核结果能证明原问题已解决。

| 案例 | 手头材料 |
| --- | --- |
| A　新内核启动 | **教学记录。** `boot-A1` 运行 `6.8.0-52-generic`。PCI 仍列出 `0000:65:00.0`，该条目未显示 `nvidia` 绑定。`lsmod` 无 `nvidia`，`modinfo -k "$(uname -r)" nvidia` 找不到模块。 |
| B　NVML 初始化失败 | **Canonical 公开示例。** `nvidia-smi` 报告版本失配，内核日志给出客户端 `570.172.08` 和运行中模块 `570.158.01`。给定材料缺少所加载 NVML 的路径及磁盘模块版本。 |
| C　首次启动 Kernel 失败 | **教学记录。** CUDA 枚举、选卡和分配成功。程序在 `launch` 报 `cudaErrorNoKernelImageForDevice`（209）。针对这个 Kernel 的产物清单只有 `sm_80` cubin，PTX 列表为空。 |
| D　同一 ELF 换一种启动环境 | **教学记录。** 文件仍为 `/opt/probe/probe-sm90`，`DT_RUNPATH=$ORIGIN/lib`，直接依赖名为 `libcudart.so.13`。两个目录有可加载的同名库，启动时新增 `LD_LIBRARY_PATH=/opt/old-app/lib`。进程报共享对象符号错误，此时尚未进入 `main`。 |
| E　同一 H100 换到 R580 | **教学变体。** 主机用 `580.65.06`。实际加载 Toolkit 13.2 GA 对应的 Runtime 和系统 CUDA 驱动库。`probe-sm90` 运行到 `output=7`。换成仅带 Toolkit 13.2 GA 生成的普通 `compute_80` PTX 的 `probe-ptx80`，应用仍能枚举和选卡，首次使用 Kernel 时返回 `cudaErrorUnsupportedPtxVersion`（222）。 |

## 参考推理

**三条路径与可见性。** 构建机的 Toolkit 用 `nvcc` 生成含主机代码与设备代码的交付件。应用进程借助 `libcudart` 和 `libcuda.so.1` 请求内核驱动访问 GPU，`nvidia-smi` 通过 NVML 走到共同的底层驱动。GSP 固件由驱动分发，由 GPU 上的 GSP 执行。运行主机的 `nvcc` 命令缺失只影响在该环境发起构建的判断。把探针的可见性变量置空后，两次探针运行首先在 CUDA 枚举处分叉，空列表使程序停在选卡之前。独立发起的 NVML 查询仍可访问 H100。 [NVIDIA Linux 驱动组件](https://download.nvidia.com/XFree86/Linux-x86_64/580.65.06/README/installedcomponents.html) · [NVIDIA GSP 固件](https://download.nvidia.com/XFree86/Linux-x86_64/580.65.06/README/gsp.html) · [CUDA 13.2.1 · CUDA_VISIBLE_DEVICES](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html#cuda-visible-devices)

**A　当前启动的模块。** PCI 枚举已成功，现有证据把故障缩到本次启动的 NVIDIA 模块准备与绑定。`modinfo` 找不到文件，先按所用打包方式查 `6.8.0-52-generic` 对应的预编译包或 DKMS 构建与安装记录。如果补充证据显示文件存在，就查同次启动日志看该模块是否被拒绝。复核时先对齐当前内核、实际模块和同一 BDF 的绑定，再重跑管理查询与探针。 [Ubuntu Server · 模块安装](https://ubuntu.com/server/docs/how-to/graphics/install-nvidia-drivers/#installing-the-kernel-modules) · [Linux 模块签名](https://docs.kernel.org/admin-guide/module-signing.html)

**B　运行中的 NVML 与模块。** NVML 初始化已到与驱动交互的步骤，两端完整版本不同。先在同一进程环境取 NVML 的实际加载路径，再看磁盘模块版本。若磁盘和系统 NVML 都是 `570.172.08`，而运行中还是 `570.158.01`，证据与更新后旧模块仍在运行相符。若磁盘与运行中模块一致，进程却从额外目录读取另一套 NVML，就应继续查库解析。复核应使用原来的启动环境和管理请求。 [Ubuntu Server · 版本失配示例](https://ubuntu.com/server/docs/how-to/graphics/install-nvidia-drivers/#driver-library-version-mismatch-error) · [glibc 动态链接器 · LD_DEBUG](https://man7.org/linux/man-pages/man8/ld-linux.8.html)

**C　当前 GPU 的 Kernel 镜像。** 选卡和分配已经通过，失败发生在首次使用设备代码。H100 是 9.0，现有的 `sm_80` cubin 无法跨到 9.x，清单也无 PTX。确认清单来自提交这个 Kernel 的文件。固定驱动、Runtime 和启动环境，只换带 `sm_90` 的同源产物。若启动检查、同步和拷回 7 均通过，就为代码目标导致失败的判断增加了单变量证据。 [NVIDIA Hopper 兼容指南](https://docs.nvidia.com/cuda/archive/12.6.0/hopper-compatibility-guide/index.html) · [CUDA 13.2.1 Runtime API · 错误码](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__TYPES.html#group__CUDART__TYPES)

**D　实际加载的共享对象。** 错误在主机动态链接阶段出现。按给定的 glibc 查找条件，额外目录排在 `RUNPATH` 前，因此预计会选中那里的 Runtime。同名只能说明它可以成为搜索结果。先取缺失符号及其版本，再用 `LD_DEBUG=libs,files` 核对报告错误的对象实际来自哪里。只去掉新增的 `LD_LIBRARY_PATH` 作对照，若加载路径回到 `/opt/probe/lib` 且程序顺利进入 `main`，就验证了装载阶段的判断。探针随后写回 7，才完成计算复核。 [glibc 动态链接器 · 路径与加载跟踪](https://man7.org/linux/man-pages/man8/ld-linux.8.html)

**E　R580 下限与 PTX JIT。** R580 达到 CUDA 13.x 的次版本兼容下限，`probe-sm90` 又有可直接装入的镜像，本行的探针结果支持这组条件下的调用可用。换成 `probe-ptx80` 后，H100 仍满足普通 PTX 的目标特性，新增条件是实际使用的 JIT 必须接受这份 PTX。222 把取证指向 PTX 的 `.version` 与实际 CUDA 驱动和 JIT 组件。可以保持 PTX 产物和 Runtime，改用第 03 篇中满足该产物要求的 `595.45.04` 驱动条件，复核启动和回读。这个对照把次版本兼容下限、设备架构匹配和 PTX JIT 支持三个条件落实到了不同的证据上。 [CUDA 次版本兼容 · PTX 限制](https://docs.nvidia.com/deploy/cuda-compatibility/latest/minor-version-compatibility.html#application-considerations-for-minor-version-compatibility) · [CUDA 13.2.1 Runtime API · 错误码](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__TYPES.html#group__CUDART__TYPES)

## CUDA 软件依赖与可用性小结

一次 CUDA 计算把 **交付件**、**运行进程**和**宿主机设备状态**接在一起。交付件声明主机依赖并携带设备代码。进程解析到能协作的库，选中设备并建立上下文，驱动才有条件取得代码、提交计算。宿主机负责让内核驱动绑定设备并提供可访问的节点。管理查询验证共享底座上的一条独立路径，完成同步与拷回才验证这次应用计算。

下一章沿这次执行继续追踪拷贝，解释主机内存、显存与多张 GPU 之间的数据经过哪些链路。第 10 章改变进程所在的环境，用同一套依赖关系判断容器实际能看到哪些设备节点和驱动库。
