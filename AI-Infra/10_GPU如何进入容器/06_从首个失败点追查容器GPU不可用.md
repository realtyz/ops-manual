# 从首个失败点追查容器 GPU 不可用

宿主机能看到 GPU，容器却用不了时，先确认失败发生在容器创建、运行时启动还是应用执行。运行时还没启动进程时，优先看设备请求和注入路径。应用已经运行时，再看设备、库、进程环境和 CUDA 调用。用同一节点和同一次尝试的证据定位首个失败点，比按报错文字直接归因更可靠。 [containerd 2.3.0 创建配置实现](https://github.com/containerd/containerd/blob/v2.3.0/internal/cri/server/container_create_linux.go) · [containerd 2.3.0 启动实现](https://github.com/containerd/containerd/blob/v2.3.0/internal/cri/server/container_start.go)

## 按阶段收集证据

1. **确认主机基线。** 记录节点、时间、目标 GPU 的 UUID 与 BDF，再记录创建请求标识或容器 ID。宿主机成功查询只验证该时点的管理通路。确认查询的是运行该容器的节点和分给它的那张卡。
2. **检查请求与配置。** 在 CRI 创建配置失败时，对照 `CDI_devices`、运行时选择、实际扫描的目录和原始创建错误。原生 CDI 的名字解析失败发生在组装 OCI 配置时。若 CRI 已有容器 ID，但低层 `create` 或 Hook 失败，应读任务启动错误。NVIDIA 包装层分支先确认实际模式是 JIT CDI、读取规格的 `cdi`，还是显式 `legacy`，再按该模式找日志。 [containerd 2.3.0 CRI 实现](https://github.com/containerd/containerd/blob/v2.3.0/internal/cri/server/container_create_linux.go) · [Toolkit 1.18.2 调试日志](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.18.2/troubleshooting.html#generating-debugging-logs)
3. **检查设备及库。** 在应用已启动的容器中，核对 OCI 注入结果和实际节点，取失败的 `openat` 路径、模式和 errno。节点存在但打开被拒时，按 PID 对齐有效 cgroup 设备策略、文件权限和安全审计。若 `nvidia-smi` 命令找不到，先查 `PATH` 和工具文件是否存在，再核对 NVML 库，同时用 CUDA 探针独立检查计算。出现库错误时查看容器挂载和加载器实际搜索记录。 [Linux cgroup v2 设备控制](https://docs.kernel.org/6.18/admin-guide/cgroup-v2.html#device-controller) · [Linux 动态链接器](https://man7.org/linux/man-pages/man8/ld.so.8.html) · [NVIDIA SMI 与 NVML](https://docs.nvidia.com/deploy/nvidia-smi/index.html)
4. **检查进程枚举与计算。** 比较 `CUDA_VISIBLE_DEVICES` 的实值及应用报告的设备 UUID。若管理查询成功而 CUDA 调用失败，记下是枚举、选卡、提交还是同步报错。选卡成功后才在首次计算报错时，再按返回值核对驱动兼容性和实际装载的库或设备代码，例如应用是否含适用于该 GPU 的 cubin，或其 PTX 是否能由当前驱动 JIT。 [CUDA 13.2.1 设备 API](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html) · [CUDA 13.2.1 错误处理](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__ERROR.html) · [NVIDIA CUDA 兼容性](https://docs.nvidia.com/deploy/cuda-compatibility/latest/) · [Hopper 设备代码兼容性](https://docs.nvidia.com/cuda/archive/12.6.0/hopper-compatibility-guide/index.html)

## 离线判读的环境与基线

以下为依据公开规范和源码构造的**教学记录**。假设节点 `node-10` 运行 Linux x86_64、cgroup v2、containerd `2.3.0`、Toolkit `1.18.2`，使用普通 runc 和非特权容器。节点有两张完整 H100，宿主机管理查询均正常，B 的 UUID 为 `GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb`，PCI BDF 为 `0000:81:00.0`，假设 B 的节点为 `/dev/nvidia1`（字符设备 `195:1`）。各条记录采用同一镜像、同一预编译 CUDA 探针及相同执行身份，驱动与探针在健康基线中兼容。`T+` 是同一教学时间轴的相对时点，`r0` 等是请求标识，`c0` 等是简写的容器标识。表中字段为摘录。

| 时间及标识 | 健康基线记录 |
| --- | --- |
| `T+00s`，`node-10` | 主机 `nvidia-smi` 按 UUID 列出 B，BDF 为 `0000:81:00.0`。 |
| `T+01s`，`r0 / c0` | CRI 选普通 runc，`CDI_devices[0].name` 为 `nvidia.com/gpu=GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb`，创建并启动成功。 |
| `T+02s`，`c0` | 容器内 NVML 列出 B，探针能加载 `libcuda.so.1`。`cudaGetDeviceCount` 返回 `cudaSuccess`、计数 `1`，序号 `0` 对应 B。 |
| `T+03s`，`c0` | `cudaSetDevice(0)`、最小 Kernel 提交检查和 `cudaDeviceSynchronize()` 均返回 `cudaSuccess`，读回值符合该探针的预期。 |

## 五条记录，五个检查点

下面的记录相互独立。除表中指明的变化外，沿用上述环境和启动基线。B 行的错误片段采用 CDI 1.1.0 源码中的措辞，C 行用系统调用和 errno 表达。D 行假设探针在调用 CUDA 前先显式检查能否加载驱动库。E 行是 CUDA API 文档允许的返回示例。 [CDI 1.1.0 名称解析错误](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/pkg/cdi/cache.go) · [Linux open 错误码](https://man7.org/linux/man-pages/man2/open.2.html) · [CUDA 13.2.1 设备 API](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)

| 教学记录 | 已知证据 | 判断及下一项检查 |
| --- | --- | --- |
| A，`T+10s`，`rA / cA` | 上层原拟使用 B。CRI 请求省略 `CDI_devices`，普通 runc 启动成功。本机规格中列有 B，`cA` 内缺 `/dev/nvidia1`。 | 本次缺少原生 CDI 注入输入。在同一镜像和运行时下补全 B 的 CDI 名重新创建，比较请求与 `linux.devices`。 |
| B，`T+20s`，`rB` | 已向普通 runc 路径提交 CDI 名，末尾误为 `…bbbbbbbbbbbc`。`CreateContainer` 失败，错误片段为 `unresolvable CDI devices`。`nvidia-ctk cdi list` 中 B 的名字末尾为 `…bbbbbbbbbbbb`。 | 失败发生在消费者解析全名时。先改用正确全名重试。正确名字仍解析失败时，再查 containerd 有效的 `cdi_spec_dirs` 及规格解析错误，区分目录配置和规格问题。 |
| C，`T+30s`，`rC / cC` | 请求 B，容器已启动。`stat` 的设备号为 `c3:1`，失败进程执行 `openat(AT_FDCWD, "/dev/nvidia1", O_RDWR)` 返回 `-1 EPERM`。 | 已到设备打开阶段。按失败 PID 查 cgroup 的有效设备规则与安全审计，再核对文件权限和实际身份。记录能定位失败的访问，具体拒绝来源还需证据。 |
| D，`T+40s`，`rD / cD` | 请求 B，NVML 列出 B。探针的 `dlopen("libcuda.so.1", RTLD_NOW)` 返回空指针，`dlerror()` 中有 `libcuda.so.1: cannot open shared object file: No such file or directory`。加载器查过缓存和默认目录。 | 先查计算通路的库解析。用 OCI 挂载、容器内文件及其链接目标对照搜索记录，区分挂载缺失与文件已存在但路径选错。 |
| E，`T+50s`，`rE / cE` | 请求 B，NVML 列出 B。探针环境中 `CUDA_VISIBLE_DEVICES` 为空，`cudaGetDeviceCount` 报 `cudaErrorNoDevice`。同一容器中取消掩码后启动新探针，枚举到 B，选卡与同步成功。 | 单独改变进程筛选就恢复计算。对齐两次探针的环境、容器 ID 和 B 的 UUID，可以把差异收窄到进程级可见性。 |

复测时固定节点、镜像、执行身份和目标 UUID，每轮只改一个条件。请求错误从新的创建结果验证，设备权限从相同进程身份下的打开结果验证，库路径从加载记录验证。完成该处复核后，再用同一探针的设备身份、提交及同步结果确认计算通路。
