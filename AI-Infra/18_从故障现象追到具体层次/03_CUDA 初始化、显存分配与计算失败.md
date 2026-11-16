
第 02 篇的终点是训练进程在容器内枚举到了 `GPU-H2-A`。但管理查询列出设备只证明注入成功，不证明应用能在卡上完成一次计算：`h2` 上训练容器的管理查询列出了 `GPU-H2-A`，应用仍然报错。CUDA（NVIDIA 的 GPU 计算平台）的 Runtime 是程序调用的用户态接口，它还需通过驱动建立计算上下文，为 GPU 维护设备内存与执行状态。用应用实际使用的用户、设备、镜像和环境测试，才能把容器交付与这条计算路径接起来。

## 一次计算从装载到结果回读

用一个单卡探针做正常对照。假设探针与待查应用使用同一已记录的 CUDA 用户态库，包含 H100 可执行的设备代码，并在新进程中选中映射到 `GPU-H2-A` 的 `cuda:1`。探针分配 4 B 设备缓冲区，提交一个 GPU Kernel（设备端函数）写入 32 位整数 7，等工作完成后拷回并比较 `output == 7`。以下检查点按本探针的程序顺序设置，训练框架可能在 import、枚举或首次算子时提前触发某些动作。CUDA API 以 13.2.1 文档为口径。 [CUDA 13.2.1 · Runtime 与驱动 API](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/driver-vs-runtime-api.html) · [CUDA 13.2.1 · 设备管理](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)

| 程序进度 | 这次成功证实的事实 | 首次失败时保留的材料 |
| --- | --- | --- |
| 进入探针，完成库装载 | 主程序和当前步骤的所需库已加载。后续按需装载仍需单独观察。 | 动态加载器消息、库的实际路径和缺失符号。执行入口前失败时，先检查镜像、搜索路径与装载的库。 |
| 设备枚举、记录 UUID、`cudaSetDevice` | 应用能看到目标卡。在此版本中，`cudaSetDevice` 会初始化所选设备上的 Runtime 状态，本探针检查它的返回值。 | 设备数、实选 UUID、返回码。核对设备权限、`libcuda.so.1` 来源、宿主机驱动及设备状态。 |
| `cudaMalloc` | 本次设备内存请求已经成功。 | 请求字节数、同卡进程与显存使用、框架分配器状态，及该调用之前是否有未完成的设备工作。 |
| Kernel 启动检查 | 主机端提交后紧跟的启动检查返回成功。 | Kernel 名称、启动参数、设备代码和错误返回。 |
| 等待本次 GPU 工作完成 | 同步点返回成功，本探针此前提交的工作已完成。 | 区分启动检查与同步返回，保留两者之间的 Stream、Kernel 与驱动事件。 |
| 回读并校验 7 | 数据已经从目标卡读回，满足本探针的预期。 | 拷贝返回值与实际结果，必要时再查输入、并发写入和数据依赖。 |

CUDA Stream 是按顺序提交设备工作的队列。主机的 Kernel 启动可先返回，GPU 稍后才执行这项工作。用 `cudaGetLastError` 记录启动检查，用 `cudaDeviceSynchronize` 等待设备工作并记录执行错误，最后回读。CUDA 的若干 API 也可能报告此前异步操作的错误。小探针用新进程且每段只提交一项工作，尽量让观察点和失败原因靠近。复杂作业若在拷贝时报错，应回查上一段没有完成证据的设备工作。 [CUDA 13.2.1 · 错误处理](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__ERROR.html) · [CUDA 13.2.1 · 异步执行](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/02-basics/asynchronous-execution.html)

## 初始化、显存压力和执行错误的区分

这些是同一探针或同一训练环境的**独立教学变体**。排查时要区分首个观测到的错误、实际失败的操作、导致它失败的条件。

### 库和访问权限改变初始化条件

一次容器更新后，管理查询仍成功，训练进程却在 CUDA 初始化报告 `cudaErrorInsufficientDriver`。采集目标进程解析到的 Runtime 与 `libcuda.so.1` 路径，并记录宿主机运行中的驱动版本。`nvidia-smi` 的 CUDA UMD Version（旧字段名为 CUDA Version）报告驱动所支持的最新 CUDA 版本。应用实际使用的 Runtime 要从进程装载的库或构建记录确认。若旧镜像在同节点、同用户、同卡成功，诊断对照还需控制镜像差异，例如仅移去新增的库搜索目录，观察实际装载路径和初始化结果。CUDA 的兼容要求还受驱动分支、Runtime 版本、所用功能和设备代码形态影响。带 PTX（供驱动即时编译的设备中间表示）的产物尤其需要核对其适用条件。 [NVIDIA SMI · CUDA UMD Version 字段](https://docs.nvidia.com/deploy/nvidia-smi/index.html) · [NVIDIA · CUDA Minor Version Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/minor-version-compatibility.html)

若同一探针能枚举设备但目标设备初始化失败，而操作系统跟踪中目标进程打开计算所需设备节点得到 `EACCES`，使用同一容器身份检查路径、设备节点权限、设备访问限制及安全策略的审计记录。设备文件存在说明路径被带进了容器，驱动操作仍需通过相应访问控制。另开特权容器会同时改变多项条件。用相同镜像和用户核对已生效的设备规则，能缩小到具体缺失的权限。 [Linux · cgroup v2 的设备访问控制](https://docs.kernel.org/6.18/admin-guide/cgroup-v2.html#device-controller)

### 小分配通过后仍可能遇到显存不足

另一次训练在新进程选卡成功后发出 6 GiB 的新分配请求，得到 `cudaErrorMemoryAllocation`。教学假定失败前没有异步设备工作，按目标 UUID 同时用 `cudaMemGetInfo` 读到可用显存约 4 GiB，4 B 探针可以成功。这个对照说明小计算能力存在，此时的显存余量不足以满足这次 6 GiB 的分配。若框架的保留量远高于存活张量的用量，进一步读取框架的分配器统计，区分同进程缓存、不可复用的空闲碎片和其他进程实际占用。将每卡工作集减小后，要重新记录峰值显存、原失败点以及整步完成。主机内存耗尽应另外根据容器终止原因和节点内存证据判断。 [CUDA 13.2.1 · 显存管理 API](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__MEMORY.html) · [PyTorch 2.9 · CUDA 显存管理](https://docs.pytorch.org/docs/2.9/notes/cuda.html#memory-management)

### 同步失败需要回看已提交的工作和设备状态

第三次探针中，启动检查成功，首次同步报错。先将该返回值限制到本进程此前提交的工作。如果相同可执行文件、输入、配置在同型号健康卡上通过，而且失败卡在同窗有驱动访问异常，优先核对设备。反过来，如果同一输入变体在多张健康卡上均触发相同非法访存，优先查应用地址、步长和对象生命周期。两类证据也可能同时出现，例如应用先遇到设备错误，错误处理又执行了一个无效操作。保留首次异常前的最后成功点，才能判断后续报错在因果链中的位置。

![h2 的 P21 训练容器内，nvidia-smi 经 NVML 查询 GPU-H2-A，CUDA 探针经 CUDA Runtime 用 cuda:1 访问同一张卡。两路都通向容器外同一个 NVIDIA 内核驱动及 H100 SXM 模组。启动检查通过，随后同步报错，未执行回读。](images/fig-18-03-01-cuda-probe-boundary.png)

图 18-03-01　容器内的管理查询已经识别目标卡，CUDA 探针的主机端启动检查也已通过。首次同步报错时，本次设备工作缺少成功完成的证据，探针未执行回读。定位时将同步返回与此前提交的工作及同窗设备日志对照。

**替代文本**　h2 的 P21 容器内，nvidia-smi 通过 NVML 列出 GPU-H2-A，CUDA 探针通过 cuda:1 访问同一张卡，两路汇合到同一个内核驱动；探针启动检查通过，随后同步报错，回读未执行。

一次受控回读让可用性有了明确的终点。训练要求的显存规模、卡间互联和持续负载仍要按原工作量复核。第 04 篇中，设备状态本身会在任务运行中改变，宿主机日志、应用错误和其他 Rank 的等待随之进入同一条时间线。
