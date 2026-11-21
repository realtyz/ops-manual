# 双卡容器 GPU 交付的综合检验

下面通过一组环环相扣的交付材料与现场记录，系统检验本章五项核心能力。作答时请先锁定每个阶段的完成条件，再逐层推导结论。

## 一、测试环境与已知条件

### 1. 节点硬件与镜像清单

服务器 `node-10` 运行 Ubuntu 24.04 LTS（cgroup v2）与 rootful `containerd 2.3.0`。节点预装 Toolkit `1.18.2` 与 R595 驱动（`595.91.07`），容器均为普通非特权容器。两张未切分 H100 的身份清单如下：

| 简称 | 宿主机管理索引 | PCI BDF | 完整 UUID | 设备节点与十进制设备号 |
| --- | ---: | --- | --- | --- |
| **GPU A** | `0` | `0000:41:00.0` | `GPU-aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa` | `/dev/nvidia0`（`195:0`） |
| **GPU B** | `1` | `0000:81:00.0` | `GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb` | `/dev/nvidia1`（`195:1`） |

镜像内的 `/opt/probe/gpu-probe` 面向 `sm_90` 编译，关键特征如下：

- `DT_NEEDED` 声明依赖 `libcudart.so.13`，`DT_RUNPATH` 为 `$ORIGIN/lib`，镜像已在 `/opt/probe/lib/` 中打包匹配版本的共享 CUDA Runtime。
- 镜像内**不含** `libcuda.so.1`、NVML、`nvidia-smi` 或任何 NVIDIA 设备节点。
- 正常健康交付会由宿主机挂入完整的计算与管理驱动文件，并提供目标卡所需的控制、UVM 与单卡设备入口，且权限与策略均放行。探针的端到端验收目标是在目标卡上写入并回读校验 `output=7`。

### 2. 原生 CDI 主例 `r0`

下面是用于手算合并的精简 CDI `1.1.0` 规格（省略 NVML 与其他辅助文件）：

```yaml
cdiVersion: "1.1.0"
kind: nvidia.com/gpu
containerEdits:
  env:
    - NVIDIA_VISIBLE_DEVICES=void
  deviceNodes:
    - path: /dev/nvidiactl
    - path: /dev/nvidia-uvm
  mounts:
    - hostPath: /usr/lib/x86_64-linux-gnu/libcuda.so.1
      containerPath: /usr/lib/x86_64-linux-gnu/libcuda.so.1
      type: bind
      options: [bind, ro]
devices:
  - name: GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb
    containerEdits:
      deviceNodes:
        - path: /dev/nvidia1
          type: c
          major: 195
          minor: 1
          permissions: rw
```

主例 `r0` 的 `handler` 为空，走默认 `runc`。containerd 已启用原生 CDI，有效目录包含这份文件。CRI 请求携带设备全名，创建与启动均成功：

```textproto
config {
  CDI_devices {
    name: "nvidia.com/gpu=GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb"
  }
}
```

### 3. 四轮独立运行时对照（R1–R4）

保持节点、镜像与目标卡（GPU B）不变，另做四轮独立启动。节点上 `nvidia` handler 的 `BinaryName` 指向 NVIDIA Runtime 包装层：

| 轮次 | handler 与包装层模式 | 设备与能力输入 | 观察到的结果 |
| --- | --- | --- | --- |
| **R1** | handler 为空（默认 `runc`） | 省略 CRI `CDI_devices`；环境变量 `NVIDIA_VISIBLE_DEVICES=<GPU B UUID>` | 容器启动，但内部没有任何 GPU 设备节点。 |
| **R2** | `nvidia`，`mode=auto` | 无 CRI CDI；`NVIDIA_VISIBLE_DEVICES=<GPU B UUID>` | 节点持久 CDI 文件临时不可读，但容器仍成功获得 GPU B。 |
| **R3** | `nvidia`，显式 `mode=legacy` | `NVIDIA_VISIBLE_DEVICES=<GPU B UUID>`，`NVIDIA_DRIVER_CAPABILITIES=utility` | Hook 执行成功，`nvidia-smi` 列出 GPU B；探针加载 `libcuda.so.1` 失败。 |
| **R4** | `nvidia`，`mode=auto` | `NVIDIA_VISIBLE_DEVICES=nvidia.com/gpu=<GPU B UUID>`（全限定 CDI 名） | 包装层按 `cdi` 模式读取持久规格，成功交付 GPU B。 |

### 4. 双卡容器的三进程观测（`c-two`）

容器 `c-two` 同时获得了 GPU A 与 GPU B。容器内管理查询列出 `0 → GPU A`、`1 → GPU B`。随后在容器内分别启动三个进程：

| 进程 | `CUDA_VISIBLE_DEVICES` | 运行记录 |
| --- | --- | --- |
| **P0** | `<GPU B UUID>,<GPU A UUID>` | 成功枚举到 2 张卡。 |
| **P1** | `<GPU A UUID>,<GPU B UUID>` | 成功枚举到 2 张卡。 |
| **P2** | `""`（空字符串） | 独立 NVML 查询仍列出 A、B；CUDA 探针报 `cudaErrorNoDevice`。 |

### 5. 创建失败现场（`rX`）

独立请求 `rX` 回到默认 `runc` 的原生 CDI 路径。设备全名与主例 `r0` 一致，但 `CreateContainer` 报 `unresolvable CDI devices`。现场核查如下：

- 宿主机执行 `nvidia-ctk cdi list` 能正常列出 GPU B 的全名；
- 规格文件位于 `/var/run/cdi/nvidia.yaml`，其中的 `kind` 与 `devices[].name` 拼写全部正确；
- containerd 当前生效的 `cdi_spec_dirs` 只有 `["/etc/cdi"]`，该目录下没有这份规格文件；
- 底层容器任务尚未创建，无容器内日志。

---

## 二、检验题目

1. **软件支路与完成条件**：分别画出 CUDA 计算与 `nvidia-smi` 查询在容器内外的完整组件路径，标明 `libcudart`、`libcuda`、NVML、设备节点和内核驱动各自来自哪里。为什么“镜像自带 Runtime”“NVML 列出 GPU B”“Kernel 启动返回成功”都不能替代最终的 `output=7`？
2. **CDI 合并与原生时序**：手算主例 `r0` 应用上述 CDI 规格后会产生哪些 OCI 字段修改，并指出由谁在哪个 CRI 阶段完成。`NVIDIA_VISIBLE_DEVICES=void` 会不会撤销此次注入？若规格中另有 GPU A 的条目并同时请求 A、B，公共修改与单卡修改如何合并？
3. **四条运行时路径判别**：逐一指出 R1–R4 各自的 OCI 修改者、是否依赖磁盘持久 CDI 文件及成功或失败的根因。R3 应优先修正哪一项配置并如何验证？要确认 R2、R4 走的不是 containerd 原生 CDI，分别应核对哪些证据？
4. **跨层身份与进程掩码**：推导 P0、P1 的 `cuda:0` 与 `cuda:1` 分别指向哪张卡，并解释 P2 报 `cudaErrorNoDevice` 是否与 NVML 列出两卡矛盾。若 P0 的 `cuda:0` 查询到 BDF `0000:81:00.0`，应如何回查到宿主机索引？
5. **创建失败定位与复核**：`rX` 的最早失败点在哪一层？为什么 `nvidia-ctk cdi list` 正常仍会触发解析失败？请给出单变量修复方案，以及从创建到端到端计算的完整复核顺序。

---

## 三、分步参考推理

> [!example]- 第 1 题参考推理：两条软件支路共享驱动底座，各自有独立交付边界
> - **CUDA 计算支路**：`gpu-probe` → 镜像 `/opt/probe/lib` 提供的 `libcudart.so.13` → 启动时由宿主机只读挂载的 `libcuda.so.1` → GPU B 所需设备节点（单卡/控制/UVM）→ 宿主机 R595 内核驱动 → 物理 GPU B。
> - **NVML 管理支路**：启动时挂入的 `nvidia-smi` → 启动时挂入的 `libnvidia-ml.so.1` → 对应设备节点 → 同一宿主机内核驱动与物理 GPU B。
> - **各里程碑不可互相替代的原因**：
>   1. 镜像自带 Runtime 只解决了上层运行时依赖，没有提供用户态驱动库 `libcuda.so.1`，也未建立设备入口与权限；
>   2. NVML 列出 GPU B 仅证实管理支路畅通，无法覆盖 `libcuda.so.1` 加载、CUDA 进程掩码或设备代码架构兼容；
>   3. Kernel 发射是异步提交，启动检查返回成功尚未确认设备端执行完成，必须等 `cudaDeviceSynchronize()` 成功并回读校验 `output=7`，整条计算交付才算闭环。

> [!example]- 第 2 题参考推理：containerd 在 `CreateContainer` 阶段合并公共与单卡修改
> - **OCI 字段手算结果**：
>   - `linux.devices[]`：加入公共的 `/dev/nvidiactl`、`/dev/nvidia-uvm`，以及 GPU B 专用的 `/dev/nvidia1`（`type: c`，`195:1`）；未显式写出的设备属性由 CDI 库从宿主机补齐。
>   - `linux.resources.devices[]`：未写 `permissions` 的控制与 UVM 节点按 CDI `1.1.0` 默认追加 `rwm` 允许规则；`/dev/nvidia1` 追加显式指定的 `rw` 允许规则（最终仍受完整规则顺序、有效 cgroup BPF、Unix 权限与 LSM 制约）。
>   - `mounts[]`：追加 `libcuda.so.1` 的只读 bind 挂载。
>   - `process.env[]`：写入 `NVIDIA_VISIBLE_DEVICES=void`。
> - **执行主体与时序**：由 **containerd CRI** 在 **`CreateContainer`** 组装配置时完成解析与合并；随后的 `StartContainer` 才交给 `shim` 和普通 `runc` 创建并启动容器。
> - **`void` 与多卡合并规则**：`NVIDIA_VISIBLE_DEVICES=void` 既不是原生 CDI 的输入，也不是 CUDA 的可见掩码，不会撤销已注入的 GPU B，它仅用于在混用旧版包装层时避免重复注入。若同时请求 GPU A 与 GPU B，顶层公共 `containerEdits` 只应用一次，再分别合并 A 与 B 各自的单卡专用条目。

> [!example]- 第 3 题参考推理：按 handler、包装层模式与输入特征精准辨认四条路径
> | 轮次 | 修改主体与持久化依赖 | 因果解释与处置重点 |
> | --- | --- | --- |
> | **R1** | 无 GPU 注入修改；未触发 CDI。 | 默认 `runc` 既没收到 CRI `CDI_devices`，也不识别 `NVIDIA_VISIBLE_DEVICES` 环境变量，因此容器无卡。补全 CRI CDI 全名即可进入原生 CDI。 |
> | **R2** | **NVIDIA Runtime 包装层**在 `create` 前内存生成（JIT CDI）；**不依赖**磁盘持久 YAML。 | `nvidia` handler + `mode=auto` + 普通 UUID 在 Toolkit `1.18.2` NVML 平台上默认走 JIT CDI，所以磁盘文件临时不可读不影响成功。 |
> | **R3** | 包装层注入 `prestart` Hook，**runc `create` 时**由 `nvidia-container-cli` 配置；**不依赖**持久 CDI。 | `NVIDIA_DRIVER_CAPABILITIES` 仅含 `utility` 漏掉 `compute`，导致 `nvidia-smi` 成功而加载 `libcuda.so.1` 失败；补齐 `compute,utility` 后复核。 |
> | **R4** | **NVIDIA Runtime 包装层**在 `create` 前读持久规格（`cdi` 模式）；**依赖**磁盘持久 YAML。 | 传入全限定 CDI 名称触发包装层切入 `cdi` 模式，由包装层而非 containerd 消费磁盘规格。 |
>
> 要确认 R2、R4 走的不是 containerd 原生 CDI，重点核对三项证据：PodSandbox 实际命中的 `handler` 为 `nvidia`、CRI 请求中 `CDI_devices` 为空，以及包装层调试日志中记录的模式解析（JIT CDI 或 `cdi`）与 OCI 修改记录。

> [!example]- 第 4 题参考推理：进程掩码只重排或清空 CUDA 视图，不影响 NVML 与硬件身份
> - **双进程映射**：
>   - **P0** 顺序为 `B, A`，因此 `cuda:0 → GPU B`、`cuda:1 → GPU A`。
>   - **P1** 顺序为 `A, B`，因此 `cuda:0 → GPU A`、`cuda:1 → GPU B`。
> - **BDF 回查**：P0 的 `cuda:0` 查出 BDF `0000:81:00.0`，对照 `node-10` 宿主机设备清单，精确锁定为 **GPU B**，对应本次宿主机管理索引 `1`（先用 UUID 确认整卡身份，再用同节点 BDF 核对物理槽位）。
> - **P2 并不矛盾**：`CUDA_VISIBLE_DEVICES=""` 仅清空了 P2 这个 CUDA 进程的可见列表，使其枚举报 `cudaErrorNoDevice`（此时应立即停止，不可继续访问 `0` 号卡）；容器本身的设备注入未受影响，独立 NVML 查询也不受该 CUDA 环境变量控制，因此仍能正常列出 A 与 B。

> [!example]- 第 5 题参考推理：生产者目录与消费者扫描目录脱节
> - **最早失败点**：**阶段 1（CRI `CreateContainer`）**，containerd 在从自己的生效扫描目录解析设备全名时失败，尚未进入低层创建和容器应用阶段。
> - **为何 `nvidia-ctk cdi list` 会给出“假绿灯”**：`nvidia-ctk cdi list` 按工具自身的默认搜索路径读到了 `/var/run/cdi/nvidia.yaml`，但 containerd 本轮生效配置里的 `cdi_spec_dirs` 仅包含 `["/etc/cdi"]`，消费者根本没扫 `/var/run/cdi`。
> - **单变量修复**：固定请求名、镜像与规格内容，只对齐目录可见性——要么将 `/var/run/cdi` 加入 containerd 的 `cdi_spec_dirs` 并重载生效，要么按统一约定将规格生成至 `/etc/cdi` 目录。
> - **端到端复核顺序**：
>   1. 原 CDI 全名提交 `CreateContainer` 返回容器 ID；
>   2. 核对生成的 OCI 配置包含 GPU B 节点、访问规则与驱动库挂载；
>   3. `StartContainer` 启动容器成功；
>   4. 容器内设备可打开、`libcuda.so.1` 实际加载；
>   5. NVML 与 CUDA 均以 UUID/BDF 对齐到 GPU B；
>   6. 探针完成选卡、Kernel 同步并回读校验 `output=7`。

---

## 四、全章知识串联与下章衔接

回顾整章，我们可以把“GPU 进入容器”浓缩成两条相互咬合的主线：

- **配置流（怎么送进去）**：上层选定物理卡 → CDI 规格声明所需节点、权限与库 → `containerd`（原生 CDI）或 NVIDIA Runtime（JIT CDI / `cdi` / legacy Hook）将声明改写进 OCI 配置 → `runc` 落地挂载命名空间与 cgroup v2 设备 BPF。
- **验证流（怎么证明能用）**：配置落地 → `openat` 穿过内核四重门禁 → `libcuda.so.1` 真正装载 → 进程用 UUID 与 BDF 对齐目标物理卡 → 同步计算校验通过。

进入下一阶段前，务必牢记一个命名边界。本章 Toolkit 示例生成的 CDI `kind` 是 `nvidia.com/gpu`。而在[第 11 章](../11_Kubernetes如何发现与分配GPU/00_内容大纲.md)中，不同组件会生成各自的类别：

- **Device Plugin 路径**：常用 `k8s.device-plugin.nvidia.com/gpu`；
- **DRA 路径**：常用 `k8s.gpu.nvidia.com/claim`。

它们都是 CDI 设备类别，与 Kubernetes 计数用的扩展资源键彼此独立。

接下来，[第 11 章 Kubernetes 如何发现与分配 GPU](../11_Kubernetes如何发现与分配GPU/00_内容大纲.md)将拉升到集群控制平面。我们将看清集群如何盘点 GPU、核算配额，并把精准选出的设备全名递交到 CRI 入口。随后的[第 12 章](../12_GPUOperator如何管理节点软件/00_内容大纲.md)至[第 14 章](../14_GPU共享、切分与隔离/00_内容大纲.md)，则会补齐节点软件托管、高速 RDMA 网络与 GPU 切分共享。

> [!info]- 参考资料
> - **CUDA 运行时、环境变量与设备 API**：[CUDA 13.2.1 Runtime 与 Driver API](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-driver-api/driver-vs-runtime-api.html)、[CUDA 13.2.1 `CUDA_VISIBLE_DEVICES` 规则](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html#cuda-visible-devices)、[CUDA 13.2.1 设备管理与同步 API](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)、[NVIDIA SMI 手册](https://docs.nvidia.com/deploy/nvidia-smi/index.html)
> - **CDI 1.1.0、containerd 2.3.0 与 Toolkit 1.18.2**：[CDI 1.1.0 规范](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/SPEC.md)、[CDI 到 OCI 转换实现（`container-edits.go`）](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/pkg/cdi/container-edits.go)、[containerd 2.3.0 CRI 创建实现（`container_create_linux.go`）](https://github.com/containerd/containerd/blob/v2.3.0/internal/cri/server/container_create_linux.go)、[Toolkit 1.18.2 CDI 支持指南](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.18.2/cdi-support.html)、[Toolkit 1.18.2 模式选择源码（`auto.go`）](https://github.com/NVIDIA/nvidia-container-toolkit/blob/v1.18.2/internal/info/auto.go)、[Toolkit 1.18.2 驱动能力说明](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.18.2/docker-specialized.html#driver-capabilities)
