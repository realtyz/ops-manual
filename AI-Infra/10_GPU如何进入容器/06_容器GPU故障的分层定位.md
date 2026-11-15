
宿主机能看到 GPU，容器却用不了时，最有效的起点不是某句报错，而是同一次尝试的时间线：CRI 是否接受了创建配置，低层运行时是否创建了任务，应用是否启动，以及应用已经越过哪个检查点。后面的证据不能倒过来证明前面的步骤成功，后面的失败也不应被直接归给最熟悉的组件。

## 先按三个阶段收窄范围

| 阶段 | 正常完成条件 | 优先核对的材料 |
| --- | --- | --- |
| CRI `CreateContainer` | containerd 接受请求并准备好容器配置，返回容器 ID。原生 CDI 的全名已解析并应用。 | 请求标识、目标 UUID、CDI 全名、有效规格目录、原始创建错误和修改后的 OCI 配置。 |
| CRI `StartContainer` 与低层任务创建 | shim 调用实际 `BinaryName`，runc `create/start` 成功；若走 legacy，Hook 已成功配置设备和文件。 | PodSandbox handler、containerd 有效 runtime 配置、NVIDIA Runtime 的实际模式、Hook/CLI 日志及任务错误。 |
| 容器应用执行 | 进程找到库、打开设备、枚举目标卡、选卡、提交并完成计算。 | 同容器挂载和设备、系统调用与 errno、动态加载记录、进程环境、CUDA API 返回值和结果校验。 |

在 containerd `2.3.0` 的这条 CRI 路径中，原生 CDI 名称解析失败属于创建配置阶段，legacy `prestart` Hook 失败则可能出现在随后的任务启动阶段。仅凭“容器创建失败”这一上层概括，无法区分两者。 [containerd 2.3.0 创建配置](https://github.com/containerd/containerd/blob/v2.3.0/internal/cri/server/container_create_linux.go) · [containerd 2.3.0 启动任务](https://github.com/containerd/containerd/blob/v2.3.0/internal/cri/server/container_start.go)

## 应用内的证据要对应到具体检查点

应用已经运行后，可按下列关系收集证据。表中是依赖关系，不要求所有程序的库加载和设备访问都呈现相同的系统调用顺序；始终以实际第一个失败调用为准。

| 检查点 | 能证明什么 | 仍不能证明什么 |
| --- | --- | --- |
| 镜像、OCI 挂载与文件存在 | 所需文件有候选来源。 | 动态链接器一定选择了它，或库与驱动兼容。 |
| `openat` 成功 | 该进程以给定模式打开了该设备入口。 | CUDA 已枚举到正确 GPU，或后续初始化会成功。 |
| `nvidia-smi` 列出 B | 这次 NVML 管理查询成功。 | `libcuda` 可加载、CUDA 进程未过滤 B、Kernel 可执行。 |
| `dlopen("libcuda.so.1")` 或程序装载成功 | 指定 CUDA 用户态对象已被解析并加载。 | 它与内核驱动及设备代码一定匹配。 |
| CUDA 枚举与 UUID/BDF 查询成功 | 该进程的某个序号可映射到 B。 | 选卡、显存分配和 Kernel 已完成。 |
| 选卡、启动检查、同步和回读成功 | 探针完成了本次预期计算。 | 其他应用、其他设备或其他版本组合也必然可用。 |

如果 `nvidia-smi` 命令找不到，先查 `PATH` 与工具文件是否存在，再核对 NVML；同时用 CUDA 探针独立检查计算。若管理查询成功而 CUDA 失败，记录是枚举、选卡、分配、启动还是同步报错。选卡成功后首次使用 Kernel 才失败时，再按第 03 章的方法检查实际驱动库、设备代码 cubin 和 PTX JIT 条件。 [CUDA 13.2.1 设备 API](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html) · [CUDA 13.2.1 错误处理](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__ERROR.html) · [NVIDIA CUDA 兼容性](https://docs.nvidia.com/deploy/cuda-compatibility/latest/)

## 一次健康基线

以下均为依据公开规范和源码构造的**教学记录**。环境为 `node-10`、Linux x86_64、cgroup v2、containerd `2.3.0`、Toolkit `1.18.2`、普通 runc 和非特权容器。B 的 UUID 为 `GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb`，BDF 为 `0000:81:00.0`，设备节点为 `/dev/nvidia1`（十进制 `195:1`）。各轮使用同一镜像、同一预编译探针及相同执行身份；健康基线中的驱动、库与设备代码兼容。

`T+` 是同一教学时间轴的相对时点，`r0`、`c0` 分别是请求和容器的简写标识。表中字段为摘录。

| 时间及标识 | 健康基线记录 |
| --- | --- |
| `T+00s`，`node-10` | 主机 `nvidia-smi` 按 UUID 列出 B，BDF 为 `0000:81:00.0`。 |
| `T+01s`，`r0 / c0` | CRI 选普通 runc，`CDI_devices[0].name` 为 `nvidia.com/gpu=GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb`；创建并启动成功。 |
| `T+02s`，`c0` | 容器内 NVML 列出 B，探针加载 `libcuda.so.1`。`cudaGetDeviceCount` 返回 `cudaSuccess`、数量 1，序号 0 对应 B。 |
| `T+03s`，`c0` | `cudaSetDevice(0)`、Kernel 启动检查和 `cudaDeviceSynchronize()` 均返回 `cudaSuccess`，读回 `output=7`。 |

基线的作用是让每次故障只改变一个已声明条件。若节点、镜像、执行身份、目标 UUID 和运行时路径同时变化，即使恢复也不能确定是哪项修改起了作用。

## 五条独立记录怎样定位首个失败点

下面每条都从相同基线独立变化，不能把它们拼成一台容器上先后发生的五个错误。B 行的错误片段采用 CDI 1.1.0 源码中的措辞，C 行保留系统调用与 errno，D 行假设探针在 CUDA 调用前显式检查驱动库，E 行采用文档允许的 CUDA 返回示例。 [CDI 1.1.0 名称解析错误](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/pkg/cdi/cache.go) · [Linux open 错误码](https://man7.org/linux/man-pages/man2/open.2.html) · [CUDA 13.2.1 设备 API](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)

| 教学记录 | 已知证据 | 最早有证据支持的判断 | 下一项区分性检查与复核 |
| --- | --- | --- | --- |
| A，`T+10s`，`rA / cA` | 上层原拟使用 B，但 CRI 请求省略 `CDI_devices`；普通 runc 启动成功。本机规格列有 B，`cA` 内缺 `/dev/nvidia1`。 | 本次没有原生 CDI 注入输入，规格存在不会改变基础 OCI 配置。 | 保持镜像与运行时不变，补全 B 的 CDI 全名重新创建，比较请求、OCI 设备与挂载；再跑探针。 |
| B，`T+20s`，`rB` | 已向普通 runc 路径提交 CDI 名，末尾误为 `…bbbbbbbbbbbc`。`CreateContainer` 失败，错误片段为 `unresolvable CDI devices`。`nvidia-ctk cdi list` 中 B 的名字末尾为 `…bbbbbbbbbbbb`。 | 失败发生在消费者解析全名时，尚未进入容器应用。 | 先改用正确全名。若仍失败，再查 containerd 有效 `cdi_spec_dirs` 与规格解析错误，区分目录和规格问题。 |
| C，`T+30s`，`rC / cC` | 请求 B，容器已启动。`stat` 的设备号为十六进制 `c3:1`，失败进程执行 `openat(AT_FDCWD, "/dev/nvidia1", O_RDWR)` 返回 `-1 EPERM`。 | 已到设备打开阶段，节点和设备号符合教学基线，这次访问被拒。 | 按失败 PID 查有效 cgroup 设备规则与安全审计，再核对文件权限和实际身份。仅凭 `EPERM` 不能唯一归因；修复后以同身份打开和探针计算复核。 |
| D，`T+40s`，`rD / cD` | 请求 B，NVML 列出 B。探针的 `dlopen("libcuda.so.1", RTLD_NOW)` 返回空指针，`dlerror()` 报告 `cannot open shared object file`；加载器已查缓存和默认目录。 | 管理支路已通过，计算支路首先卡在 `libcuda` 的解析。 | 用 OCI 挂载、容器内文件及链接目标对照搜索记录，区分未挂载与路径未选中。只修正对应条件，再核对实际加载路径和 `output=7`。 |
| E，`T+50s`，`rE / cE` | 请求 B，NVML 列出 B。探针环境中 `CUDA_VISIBLE_DEVICES` 为空，`cudaGetDeviceCount` 报 `cudaErrorNoDevice`。同容器取消掩码后的新探针枚举到 B，选卡与同步成功。 | 两次运行只改变进程筛选，差异已收窄到 CUDA 进程级可见性。 | 对齐两次探针的环境、容器 ID 和 B 的 UUID，确认容器注入未变；保留新进程的完整计算结果作为复核。 |

如果换成 NVIDIA Runtime 路径，还要在最早阶段多确认一项：handler 是否真的选中包装层，包装层实际解析为 JIT CDI、`cdi` 还是 legacy。legacy Hook 的任务启动错误不应被拿去盲目刷新 containerd 的 CDI 目录；JIT CDI 成功也不证明节点上的持久规格被消费。 [Toolkit 1.18.2 调试日志](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.18.2/troubleshooting.html#generating-debugging-logs) · [Linux cgroup v2 设备控制](https://docs.kernel.org/6.18/admin-guide/cgroup-v2.html#device-controller)

## 修复要复核到原目标，而不是只让报错消失

每轮固定节点、镜像、执行身份、目标 UUID 与无关配置，只改变一个已定位条件。请求错误看新的创建结果，设备权限看相同身份的打开结果，库路径看加载器实际选择，进程掩码看新进程的枚举身份。局部复核通过后，再让同一探针完成选卡、启动检查、同步和 `output=7`，确认下游没有第二个独立问题。

第 99 篇将换一组组合材料，要求不看上述结论也能重建整条交付链，并判断现有证据在何处失去区分力。
