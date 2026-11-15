
在教学基线中，用 CUDA Toolkit 13.2 GA 构建的 `probe-sm90` 能在 `595.45.04` 驱动下完成一次写入。把它交给另一台 H100 主机，管理工具却显示 `CUDA Version: 13.0`。这时需要判断两件事。驱动能否支持实际加载的 Runtime 和应用调用的功能？驱动能否从交付件为当前 GPU 取得可执行的 Kernel？先固定驱动看设备代码，再固定产物看驱动变化。

## 计算能力与 cubin、PTX 的装载规则

**计算能力（compute capability）** 用主版本和次版本标识 NVIDIA GPU 的架构及功能集。例如 H100 属于 9.0。编译时可以针对相应功能集生成设备代码。一个主机可执行文件或共享库中的 **fatbin** 可以收纳不同架构的设备二进制、供即时编译的中间代码，或同时收纳两者。驱动使用某个 Kernel 时，从它的候选代码中取得适用镜像。 [NVIDIA GPU 计算能力表](https://developer.nvidia.com/cuda/gpus) · [CUDA 13.2.1 编程指南 · CUDA 平台](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/01-introduction/cuda-platform.html)

- **cubin（CUDA binary）** 是针对真实 GPU 架构生成的设备二进制。例如 `sm_90` 表示为标准 9.0 目标生成。对于这里的普通目标，GPU 与 cubin 必须有相同的计算能力主版本，且 GPU 次版本至少达到 cubin 的次版本。因此 `sm_80` 可用于 8.6，`sm_86` 无法用于 8.0，9.0 则无法直接装入 8.x 的普通 cubin。
- **PTX（Parallel Thread Execution）** 是面向虚拟 GPU 架构的中间指令表示。普通目标 `compute_80` 生成的 PTX 可用于满足其 8.0 功能要求的 GPU，包括 H100。驱动需在主机侧完成 JIT（Just-In-Time，即时编译），把 PTX 转成适合实际 GPU 的设备二进制。除了目标功能集，还要满足驱动 JIT 能读懂该 PTX 的指令集版本，且 JIT 可用并已启用。

按默认规则，驱动找到适用的 cubin 就直接使用。缺少适用 cubin 时，再查有无可由当前驱动编译的 PTX。`sm_90a` 和 `compute_90a` 这类架构特化目标使用对应架构才有的功能，须按特化目标的规则判断。下面的案例使用无后缀的普通目标。 [NVIDIA Hopper 兼容指南 · Application Compatibility](https://docs.nvidia.com/cuda/archive/12.6.0/hopper-compatibility-guide/index.html#application-compatibility-on-hopper-architecture)

## 同一 Kernel 在 H100 上的三种构建结果

保留第 02 篇的 `probe.cu`、共享 Runtime 选项和 Toolkit 13.2 GA。`-gencode` 的 `arch` 指定编译时采用的虚拟架构，`code` 指定要放进产物的代码。`code=sm_…` 保存 cubin，`code=compute_…` 保留 PTX。假设同一张未切分 H100、`595.45.04` 驱动和应用库均可用，驱动可按默认规则选择代码，且这一环境的 JIT 已启用并支持产物的 PTX 版本。 [CUDA 13.2 GA · NVCC Code Generation Options](https://docs.nvidia.com/cuda/archive/13.2.0/cuda-compiler-driver-nvcc/index.html) · [NVIDIA Hopper 兼容指南](https://docs.nvidia.com/cuda/archive/12.6.0/hopper-compatibility-guide/index.html)

| 产物 | `-gencode` 的参数 | 设备代码判读 |
| --- | --- | --- |
| `probe-sm90` | `arch=compute_90,code=sm_90` | 只有 `sm_90` cubin，和 H100 的 9.0 相符，直接装入并进入计算。 |
| `probe-sm80` | `arch=compute_80,code=sm_80` | 只有 `sm_80` cubin。H100 的主版本为 9，无法装入 8.x 镜像，产物也缺少可供 JIT 的 PTX。 |
| `probe-ptx80` | `arch=compute_80,code=compute_80` | 只有面向 `compute_80` 的 PTX。H100 满足目标能力，驱动 JIT 在主机上生成 H100 可执行的设备代码，再交给 GPU。 |

![三组独立的 H100 9.0 场景。probe-sm90 的 sm_90 cubin 直接装入，probe-sm80 的 sm_80 cubin 在主机侧被阻断，probe-ptx80 的 compute_80 PTX 经主机侧 CUDA 驱动 JIT 后装入。](images/fig-03-03-01-h100-device-code.png)

**图 03-03-01**　各列分别表示在同一硬件条件下运行一个独立产物。右列成立还要求驱动 JIT 支持产物中的 PTX 指令集版本。箭头表示代码处理或装入。

第 02 篇已给出 `probe-sm90` 的构建命令。下面是另外两次构建和产物检查，检查设备代码本身无需运行探针。对每个文件都分别列出设备 ELF 和 PTX，才能同时验证产物包含什么、缺少什么。 [CUDA 13.2.1 · cuobjdump](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-binary-utilities/index.html)

```bash
nvcc --cudart=shared -gencode=arch=compute_80,code=sm_80 probe.cu -o probe-sm80
nvcc --cudart=shared -gencode=arch=compute_80,code=compute_80 probe.cu -o probe-ptx80

for artifact in probe-sm90 probe-sm80 probe-ptx80; do
    cuobjdump --list-elf "$artifact"
    cuobjdump --list-ptx "$artifact"
done
cuobjdump --dump-ptx probe-ptx80
```

对照表里的预期读输出。`--list-elf` 列的是 fatbin 中的设备 ELF。`--dump-ptx` 则可以看到 `.target` 和 `.version`。本例 `compute_80` 的 PTX 中，`.target sm_80` 给出目标所需的架构特性，`.version` 给出 PTX 指令集版本。如果 Kernel 来自按需加载的 `.so`，要检查那个库和对应 Kernel 的镜像。主程序的清单只覆盖主程序自身。 [CUDA Binary Utilities · cuobjdump](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-binary-utilities/index.html) · [CUDA 13.2 GA · PTX ISA](https://docs.nvidia.com/cuda/archive/13.2.0/parallel-thread-execution/index.html)

设备代码有两个常用的取证方向。`cudaErrorNoKernelImageForDevice`（209）表示当前设备缺少适用的 Kernel 镜像，先查 GPU 与产物目标。`cudaErrorUnsupportedPtxVersion`（222）表示驱动无法处理该工具链生成的 PTX，先查这份 PTX 和进程实际使用的驱动 JIT。这里的错误含义来自 API 文档，三个产物的运行结果为教学推演。 [CUDA 13.2.1 Runtime API · 错误码](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__TYPES.html#group__CUDART__TYPES)

## 驱动向后兼容与 CUDA 次版本兼容

驱动还要支持应用加载的 Runtime 和实际调用的功能。**驱动向后兼容**让较新的驱动继续服务较早 Toolkit 构建的应用。比如支持 CUDA 13.2 的驱动可运行按 CUDA 12.x 构建的应用，前提是驱动支持当前 GPU 与平台，应用的旧版库和适用设备代码也已就绪。CUDA 驱动支持某个 API 版本，仍需应用提供自己所需的库。 [CUDA 13.2 GA Release Notes · CUDA Driver](https://docs.nvidia.com/cuda/archive/13.2.0/cuda-toolkit-release-notes/index.html#cuda-driver)

**次版本兼容（minor version compatibility）**提供另一条路径。对于同一 CUDA 主版本系列，较新的 Runtime 可在满足最低要求的较早驱动上使用受限的功能集。读 CUDA 13.2 GA Release Notes 时，两张表回答的是两个问题。表 2 给出 13.x 的最低驱动分支 R580，表 3 给出 Toolkit 13.2 GA 对应的 Linux x86_64 驱动版本 `595.45.04`。前者决定能否考虑次版本兼容，后者给出该 Toolkit 的常规配对基线。 [CUDA 13.2 GA Release Notes · 表 2 与表 3](https://docs.nvidia.com/cuda/archive/13.2.0/cuda-toolkit-release-notes/index.html#cuda-driver) · [CUDA 次版本兼容](https://docs.nvidia.com/deploy/cuda-compatibility/latest/minor-version-compatibility.html)

固定 `probe-sm90` 和它的 13.2 GA 共享 Runtime，运行主机仍用 x86_64 Linux 和 H100。若主机采用 `580.65.06`，虽然管理工具可以显示 `CUDA Version: 13.0`，该驱动已达到 R580 下限。探针有 `sm_90` cubin 可直接装入，只要所用功能得到该驱动支持，就可走次版本兼容。若调用依赖较新驱动的功能，可能遇到 `cudaErrorCallRequiresNewerDriver`。若换成仅带 13.2 GA 生成的 PTX 的 `probe-ptx80`，又把 JIT 放回了必经路径，R580 下限本身给不出 PTX JIT 可用的保证。 [CUDA 次版本兼容 · Application Considerations](https://docs.nvidia.com/deploy/cuda-compatibility/latest/minor-version-compatibility.html#application-considerations-for-minor-version-compatibility)

## Linux CUDA 前向兼容包

若驱动降到 `570.124.06`，它属于 R570，低于 13.x 次版本兼容下限。NVIDIA 的 **前向兼容（forward compatibility）**适用于符合条件的 Linux 系统和 GPU。兼容包提供较新的 CUDA 用户态驱动库（包括 `libcuda` 与 PTX JIT 等指定组件），应用进程加载后仍经基础驱动的内核模块访问 GPU。它与次版本兼容作用在不同接口上。前者让新版 CUDA 用户态驱动与较早内核驱动在支持的组合内工作，后者让较新的 Runtime 使用较早的 CUDA 驱动。 [NVIDIA CUDA 前向兼容 · 包内容和功能例外](https://docs.nvidia.com/deploy/cuda-compatibility/latest/forward-compatibility.html)

继续查官方前向兼容矩阵，`cuda-compat-13-2` 行与 `570+ (CUDA 12.8)` 列交叉为 `C`，表示该表支持这个兼容包与基础分支的组合。对 H100 教学场景，接下来仍要确认平台条件、进程是否实际加载兼容包中的库，以及应用所需功能是否要求更新内核驱动。安装包只放置这些 CUDA 用户态文件，具体启动环境还要能解析到它们。也要照常核对 Runtime 和 Kernel 的设备代码。 [NVIDIA CUDA 前向兼容 · 支持矩阵](https://docs.nvidia.com/deploy/cuda-compatibility/latest/forward-compatibility.html#deployment-considerations-for-forward-compatibility)

## 交付清单中的版本与代码证据

| 要判断的条件 | 取证方式 |
| --- | --- |
| 目标 GPU 与运行中的驱动 | 按 BDF 或 UUID 确认型号，查计算能力。记录实际加载的内核模块版本和进程使用的 CUDA 用户态驱动库，前向兼容场景尤其要看库来源。 |
| 应用构建与实际依赖 | 交付清单记录构建 Toolkit、Runtime 及额外库、编译目标。共享库的实际路径从进程取，静态编入的组件从构建记录取。`libcudart.so.13` 只提供共享库接口主版本的线索。 |
| 使用到的设备代码 | 对真正提交失败 Kernel 的产物检查 cubin、PTX 或运行时生成代码的来源。主程序、扩展与计算库可能各有自己的要求。 |

Toolkit 组件从 CUDA 11.0 起独立编号。还需区分工具链支持的构建目标与驱动支持的运行目标。CUDA Toolkit 13.0 移除了对 Maxwell、Pascal、Volta 的离线编译与库支持。既有产物能否继续运行，要看具体驱动是否仍支持该卡、所需库是否可用、产物是否含有适用代码。运行时库的实际路径与内核模块的实际版本，正是第 04 篇要从 Linux 现场取得的证据。 [CUDA 13.2 GA · 组件版本](https://docs.nvidia.com/cuda/archive/13.2.0/cuda-toolkit-release-notes/index.html#cuda-toolkit-major-components) · [CUDA 13.0 Release Notes · Deprecated Architectures](https://docs.nvidia.com/cuda/archive/13.0.0/cuda-toolkit-release-notes/index.html#deprecated-architectures)
