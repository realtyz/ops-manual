
下面用一组新的组合材料检验本章的五项目标。全部配置和记录都是**教学推演**，无需 GPU 实验环境。做题时先保留每个阶段的完成条件，再判断证据能支持到哪里。

## 节点、镜像与设备描述

`node-10` 使用 Linux x86_64、rootful containerd `2.3.0`、cgroup v2、普通非特权容器和 Toolkit `1.18.2`。两张完整 H100 的驱动与硬件均健康。PCI BDF、UUID 和本次主机管理索引如下；设备节点是本轮另行核对的教学假设，不由索引推算。

| 卡   | 主机管理索引 | PCI BDF        | UUID 简称                                    | 设备节点与十进制设备号            |
| --- | -----: | -------------- | ------------------------------------------ | ---------------------- |
| A   |      0 | `0000:41:00.0` | `GPU-aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa` | `/dev/nvidia0`，`195:0` |
| B   |      1 | `0000:81:00.0` | `GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb` | `/dev/nvidia1`，`195:1` |

镜像 `/opt/probe/gpu-probe` 是第 03 章同类的 `sm_90` 探针，`DT_NEEDED` 声明 `libcudart.so.13`，`DT_RUNPATH` 为 `$ORIGIN/lib`，匹配 Runtime 已放在 `/opt/probe/lib/`。镜像不含 `libcuda.so.1`、NVML、`nvidia-smi` 或 NVIDIA 设备节点。完整健康交付会由宿主机挂入匹配的计算与管理文件，并提供目标卡所需的控制、UVM 和单卡入口；文件权限、设备规则与安全策略均满足要求。探针最终应在目标卡上写回并校验 `output=7`。

下面这份 CDI YAML 只用于手算配置合并，省略了 NVML、工具和其他驱动依赖：

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

原生主例 `r0` 的 handler 为空，因此使用 `default_runtime_name = "runc"`。请求含：

```textproto
config {
  CDI_devices {
    name: "nvidia.com/gpu=GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb"
  }
}
```

containerd 的 CDI 已启用，有效规格目录包含这份文件。`CreateContainer` 与 `StartContainer` 均成功。

## 四轮独立运行时对照

另做四轮相互独立的启动。除表中声明外，镜像、节点、目标 B、执行身份与健康条件都相同。包装层平台为本章的 NVML 平台，`nvidia` handler 对应的 `BinaryName` 是 `/usr/bin/nvidia-container-runtime`。

| 轮次 | handler 与模式 | 设备输入 | 已知结果 |
| --- | --- | --- | --- |
| R1 | handler 为空，默认普通 runc | 省略 CRI `CDI_devices`；进程环境有 `NVIDIA_VISIBLE_DEVICES=<B 的完整 UUID>` | 容器启动，根文件系统仍无 B 的设备节点。 |
| R2 | `nvidia`，`mode=auto` | 无 CRI CDI；`NVIDIA_VISIBLE_DEVICES=<B 的完整 UUID>` | 包装层按 JIT CDI 处理；本机持久规格临时不可读，但容器仍获得 B。 |
| R3 | `nvidia`，显式 `mode=legacy` | `NVIDIA_VISIBLE_DEVICES=<B 的完整 UUID>`，`NVIDIA_DRIVER_CAPABILITIES=utility` | Hook 执行成功，`nvidia-smi` 列出 B；探针显式加载 `libcuda.so.1` 失败，其他库搜索条件正常。 |
| R4 | `nvidia`，`mode=auto` | `NVIDIA_VISIBLE_DEVICES=nvidia.com/gpu=<B 的完整 UUID>`，使用全限定 CDI 名 | 包装层按 `cdi` 模式读取持久规格，而不是由 containerd CRI 消费。 |

## 双卡容器中的三个进程

容器 `c-two` 已通过同一轮启动获得 A、B，管理查询示例按 `0 → A`、`1 → B` 列出两卡。三个进程分别启动，库与权限均正常：

| 进程 | `CUDA_VISIBLE_DEVICES` | 其他记录 |
| --- | --- | --- |
| P0 | `B,A`（实际值为两个完整 UUID） | 已成功枚举两张卡。 |
| P1 | `A,B` | 已成功枚举两张卡。 |
| P2 | 空字符串 | 独立 NVML 查询仍列出 A、B；CUDA 探针报 `cudaErrorNoDevice`。 |

## 一份新的创建失败记录

最后的 `rX` 与 R3、R4 无关，回到普通 runc 的原生 CDI 请求，名称与主例完全相同。本次 `CreateContainer` 报 `unresolvable CDI devices`。已知：

- `nvidia-ctk cdi list` 能在本机列出 B 的全名；
- 规格文件位于 `/var/run/cdi/nvidia.yaml`，其中 `kind` 与 `devices[].name` 正确；
- containerd 本轮有效配置只有 `cdi_spec_dirs = ["/etc/cdi"]`，该目录没有这份规格；
- 低层任务尚未创建，容器内没有应用记录。

## 检验任务

1. **依赖与完成条件。** 根据镜像和健康交付条件，分别画出 CUDA 计算与 `nvidia-smi` 查询的路径，标明 `libcudart`、`libcuda`、NVML、设备节点和内核驱动各由哪里提供。为什么“镜像含 Runtime”“NVML 列出 B”“Kernel 已提交”都不能单独替代 `output=7`？
2. **CDI 合并与原生时序。** 对 `r0` 逐项写出公共与 B 专用 `containerEdits` 应产生的 OCI 修改，并指出由谁、在哪个 CRI 阶段应用。`NVIDIA_VISIBLE_DEVICES=void` 会不会撤销这次 CDI 注入？若规格中另有 A 的条目并同时请求 A、B，公共修改与各卡专用修改怎样合并？
3. **辨认实际运行时路径。** 分别解释 R1—R4 的修改者、是否依赖持久 CDI 规格及主要失败或成功原因。R3 应优先改变哪项输入，怎样验证修复？R2 与 R4 都要查哪些证据，才能确认不是 containerd 原生 CDI？
4. **跨层设备身份。** 推导 P0、P1 的 `cuda:0/1` 分别对应哪张卡，P2 为何没有有效 CUDA 序号。若 P0 的序号 0 返回 BDF `0000:81:00.0`，如何回查主机管理索引？P2 的 NVML 结果是否与 CUDA 结果矛盾？
5. **定位创建失败。** `rX` 的最早失败点在哪里？`nvidia-ctk cdi list` 的结果为什么不足以证明 containerd 能解析该名称？给出下一项核对、一个单变量修复方案，以及从创建到实际计算的复核顺序。

## 参考推理

### 1. 两条软件支路共享驱动底座，但各有交付条件

计算路径为：`gpu-probe` → 镜像 `/opt/probe/lib` 中的 `libcudart.so.13` → 启动时从宿主机挂入的 `libcuda.so.1` → B 所需设备节点 → 宿主机 NVIDIA 内核驱动 → GPU B。管理路径为：注入的 `nvidia-smi` → 注入的 NVML → 相应设备节点 → 同一内核驱动与 GPU。两条支路都要满足设备访问条件，但加载的工具和库不同。 [CUDA Runtime 与 Driver API](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-driver-api/driver-vs-runtime-api.html) · [NVIDIA SMI 与 NVML](https://docs.nvidia.com/deploy/nvidia-smi/index.html)

镜像含 Runtime 只解决应用依赖的一部分，既没有提供本例的 `libcuda`，也没有建立设备访问。NVML 列出 B 只证明管理支路成功，不能证明计算库、CUDA 进程筛选或设备代码。Kernel 提交与设备完成是不同检查点；探针还需同步成功、拷回并校验 7，才覆盖本例的计算目标。 [CUDA 13.2.1 同步与错误处理](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)

### 2. containerd 在配置阶段展开公共与专用修改

请求 B 时，公共部分贡献 `/dev/nvidiactl`、`/dev/nvidia-uvm`、示例 `libcuda` 只读 bind 挂载和 `NVIDIA_VISIBLE_DEVICES=void`；B 专用部分贡献 `/dev/nvidia1`。CDI 库将这些描述转换为 OCI `linux.devices`、相应 `linux.resources.devices` 访问项、`mounts` 与 `process.env`。公共设备节点未写 `permissions`，按 CDI 1.1.0 实现默认追加 `rwm` 允许项；B 节点则显式追加 `rw` 允许项。这些只是 OCI 访问规则输入，完整规则顺序、有效 cgroup 策略、Unix 文件权限与 LSM 安全策略仍须独立满足。containerd CRI 在 `CreateContainer` 组装配置时完成解析和修改，随后 `StartContainer` 才让 shim/runc 创建并启动任务。裁剪 YAML 没列出的 NVML 与其他文件来自题设的完整健康条件，不能从这张摘录中凭空推出。 [CDI 1.1.0 规范](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/SPEC.md) · [CDI 到 OCI 的转换](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/pkg/cdi/container-edits.go) · [containerd 2.3.0 CRI 实现](https://github.com/containerd/containerd/blob/v2.3.0/internal/cri/server/container_create_linux.go)

`void` 是写入进程环境的值，不是原生 CDI 的选择输入，也不是 `CUDA_VISIBLE_DEVICES` 掩码；它不会撤销已经由全名展开的 OCI 修改。若同时请求 A、B，公共修改应用一次，A、B 的专用修改分别加入。两张卡的入口和最终 UUID/BDF 仍须分别验证。

### 3. 先看 handler，再看模式和设备输入

| 轮次 | 推导 | 修复或确认重点 |
| --- | --- | --- |
| R1 | 默认 runc 不消费 CRI CDI，因为请求已省略；它也不会把 `NVIDIA_VISIBLE_DEVICES` 自动转换成 NVIDIA 注入。节点有规格仍不能改变结果。 | 保持普通 runc 时，补回正确的 CRI CDI 全名，再比较 OCI 与设备。 |
| R2 | handler 选中 NVIDIA 包装层；在指定版本、NVML 平台和普通 UUID 选择器下，`auto` 解析为 JIT CDI。包装层在内存生成并应用描述，所以持久规格不可读不妨碍这条给定路径。 | 查 handler、包装层实际模式与修改后的 OCI；成功不能用来证明节点持久规格已被 containerd 消费。 |
| R3 | handler 选中包装层，legacy 在 runc `create` 阶段经 `prestart` Hook 和 CLI 配置。输入只有 `utility`，与 NVML 可用但计算库缺失的记录一致。 | 保持其他条件不变，加入 `compute`，确认 Hook 成功、`libcuda` 实际加载，再完成探针同步和 `output=7`。 |
| R4 | 全限定 CDI 名使包装层走 `cdi` 模式，由包装层读取持久规格并在低层 `create` 前应用修改。 | 持久规格必须对包装层可用；查包装层的名称解析和修改记录，而不是只查 CRI `CDI_devices`。 |

R2、R4 都经过 NVIDIA 包装层，r0 的修改者才是 containerd CRI。判断依据是本次 handler、实际模式、设备名形式、修改后的 OCI 和阶段日志，不能只看是否安装了 Toolkit。JIT 与模式选择以 Toolkit `1.18.2` 和题设平台为限，legacy 在该版本仍受支持但已弃用。 [Toolkit 1.18.2 CDI 支持](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.18.2/cdi-support.html) · [Toolkit 1.18.2 模式选择](https://github.com/NVIDIA/nvidia-container-toolkit/blob/v1.18.2/internal/info/auto.go) · [Toolkit 1.18.2 驱动功能](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.18.2/docker-specialized.html#driver-capabilities)

### 4. 序号只在各自枚举范围内有效

P0 的列表是 B、A，所以 `cuda:0 → B`、`cuda:1 → A`；P1 恰好相反，`cuda:0 → A`、`cuda:1 → B`。容器注入集合没变，变的是每个 CUDA 进程的列表顺序。P0 序号 0 的 BDF `0000:81:00.0` 在主机清单中对应 B，因此可回查到本次主机管理索引 1；UUID 用来确认身份，BDF 用来核对同节点物理位置。

P2 的空掩码使 CUDA 枚举报 `cudaErrorNoDevice`，没有可调用的有效设备序号，不应继续读取序号 0。独立 NVML 查询不随这个 CUDA 进程变量重排或清空，仍列出 A、B，两者不矛盾。 [CUDA 13.2.1 可见设备](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/05-appendices/environment-variables.html#cuda-visible-devices) · [CUDA 13.2.1 设备 API](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)

### 5. 生产者能列出名称，不等于消费者扫描了同一份规格

`rX` 的请求名与文件内容都正确，但 containerd 的有效目录只有 `/etc/cdi`，而文件在 `/var/run/cdi`。最早失败点是 containerd 在 `CreateContainer` 阶段从自己的有效规格集合中解析名称；低层任务和应用尚未开始，所以此时查容器内 CUDA 日志没有区分力。

`nvidia-ctk cdi list` 只证明该工具按自己的查找条件能列出 B，不能证明 containerd 的配置、缓存和本次消费者视图相同。下一项应把 containerd 实际加载的 `cdi_spec_dirs`、目录内容与创建错误对齐。单变量修复可以只改变消费者可见性：按所选部署方式把同一份有效规格放入已扫描目录，或调整 containerd 的有效目录并确认它已加载该规格，其余请求与镜像不变。

复核顺序是：同名请求通过 `CreateContainer` → 修改后的 OCI 含 B 的节点、规则和库挂载 → `StartContainer` 成功 → 容器内设备可打开、`libcuda` 实际加载 → NVML 与 CUDA 都以 UUID/BDF 对上 B → 同步并回读 `output=7`。任何一步失败，都停在该处继续取证，不把后面的成功假定提前写入结论。

## 从容器访问接回 Kubernetes 设备分配

本章把一次 GPU 交付拆成了两组关系。第一组是**配置怎样形成**：上层选定设备，CDI 描述设备需要的修改，containerd 或 NVIDIA Runtime 应用修改，runc 建立实际环境。第二组是**应用怎样证明可用**：库实际加载、设备实际打开、UUID/BDF 对上目标卡，最后由同步与结果校验确认计算完成。NVML 查询、CDI 文件和运行时登记各自只覆盖其中一个检查点。

还要保持名称边界：本章 Toolkit 示例的 CDI kind 是 `nvidia.com/gpu`；第 11 章 Device Plugin 示例使用 `k8s.device-plugin.nvidia.com/gpu`，DRA 示例使用 `k8s.gpu.nvidia.com/claim`。它们是不同生成路径下的 CDI 设备类别，不是 Kubernetes 用来计数的扩展资源键。第 11 章将向前接上设备发现、Node 资源、Pod 调度与具体选卡，解释这些全名为什么会出现在创建请求中。
