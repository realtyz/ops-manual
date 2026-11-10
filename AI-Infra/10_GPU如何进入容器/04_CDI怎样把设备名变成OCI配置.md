# CDI 怎样把设备名变成 OCI 配置

节点上有 `/var/run/cdi/nvidia.yaml`，普通容器仍可能没有 GPU。CDI（Container Device Interface）为运行时提供从设备全名到容器配置修改的映射。上层先决定给容器哪张卡，运行时再按这个名字准备设备访问。选卡、资源计数和避免重复分配由上层管理，CDI 文件只描述给定设备怎样接入容器。 [CDI 1.1.0 规范](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/SPEC.md)

## 生成、请求、解析和应用

以 Linux x86_64、containerd `2.3.0`、Toolkit `1.18.2` 和普通 runc 为例。假设已选定完整 GPU B，UUID 为 `GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb`。下面是按 CDI 1.1.0 编写的**教学摘录**，库路径和设备号均为假设。实际生成结果还可能包括控制及 UVM 节点、NVML 库、工具和辅助 Hook。应按文件实际声明的 `cdiVersion` 和所用字段检查消费者兼容性。containerd `2.3.0` 和 Toolkit `1.18.2` 均引用 CDI `1.1.0` 库，生成文件的 `cdiVersion` 由实际规格决定。 [CDI 版本规则](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/SPEC.md#version) · [containerd 2.3.0 依赖](https://github.com/containerd/containerd/blob/v2.3.0/go.mod) · [Toolkit 1.18.2 依赖](https://github.com/NVIDIA/nvidia-container-toolkit/blob/v1.18.2/go.mod) · [Toolkit 公共设备发现实现](https://github.com/NVIDIA/nvidia-container-toolkit/blob/v1.18.2/pkg/nvcdi/common-nvml.go)

```yaml
cdiVersion: "1.1.0"
kind: nvidia.com/gpu
containerEdits:
  env:
    - NVIDIA_VISIBLE_DEVICES=void
  deviceNodes:
    - path: /dev/nvidiactl
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

`kind` 和 `devices[].name` 组成全名 `nvidia.com/gpu=GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb`。请求这个名字时，公共的 `containerEdits` 随规格应用一次，另加 B 对应的修改。上例公共部分提供控制节点和计算驱动库，B 的条目提供 `/dev/nvidia1`。原生 CDI 以全名为注入输入，上例的 `NVIDIA_VISIBLE_DEVICES=void` 只是应用到容器环境的字段。 [CDI 1.1.0 规范](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/SPEC.md) · [CDI 1.1.0 规格合并实现](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/pkg/cdi/cache.go)

下面从 `CreateContainerRequest` 中只摘出设备请求部分，用 CRI protobuf 文本格式表示。`CDI_devices` 是 proto 字段名，对应 containerd Go 源码中的 `CDIDevices`。 [CRI v0.36.0 接口](https://github.com/kubernetes/cri-api/blob/v0.36.0/pkg/apis/runtime/v1/api.proto)

```textproto
config {
  CDI_devices {
    name: "nvidia.com/gpu=GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb"
  }
}
```

在此路径上，Toolkit 生成描述，containerd CRI 消费描述。containerd 2.0 起默认开启 CDI。`2.3.0` 的 `version = 3` 配置在 `plugins."io.containerd.cri.v1.runtime"` 下默认使用 `enable_cdi = true`，规格目录为 `cdi_spec_dirs = ["/etc/cdi", "/var/run/cdi"]`。CRI 收到设备全名后，在组装 OCI 配置时解析并应用相应修改。shim 后续可直接调用 runc，runc 根据 OCI 配置执行挂载、设备和进程设置。`2.3.0` 在显式禁用 CDI 时会拒绝带非空 `CDIDevices` 的请求。 [containerd 2.0 默认值变更](https://github.com/containerd/containerd/blob/v2.0.0/docs/containerd-2.0.md#cdi-is-now-enabled-by-default) · [containerd 2.3.0 配置](https://github.com/containerd/containerd/blob/v2.3.0/docs/cri/config.md) · [containerd 2.3.0 CRI 实现](https://github.com/containerd/containerd/blob/v2.3.0/internal/cri/server/container_create_linux.go)

| OCI 字段 | 无 CDI 请求时的基础配置 | 应用上例后增加的修改 |
| --- | --- | --- |
| `linux.devices` | 基础容器设备 | 控制节点与字符设备 `/dev/nvidia1`。 |
| `linux.resources.devices` | 基础设备规则 | 控制节点的规则，以及设备号 `195:1` 的 `rw` 允许项。 |
| `mounts` | 基础挂载 | 示例 `libcuda.so.1` 的只读 bind 挂载。 |
| `process.env` | 镜像与请求中的环境 | 应用 `NVIDIA_VISIBLE_DEVICES=void`。 |

CDI 也可以将辅助 Hook 加入 OCI 的 `hooks`，例如执行容器库缓存更新。判断注入路径时，应区分这种辅助 Hook 与 NVIDIA Runtime legacy 模式的设备配置 Hook。 [CDI 到 OCI 的转换](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/pkg/cdi/container-edits.go) · [NVIDIA 驱动文件及辅助 Hook](https://github.com/NVIDIA/nvidia-container-toolkit/blob/v1.18.2/pkg/nvcdi/driver-nvml.go)

## 怎样确认节点上的规格被使用了

Toolkit 从 `1.12.0` 起支持生成 CDI 规格。自 `1.18.0` 起，具备相应 systemd 单元的安装可用 `nvidia-cdi-refresh` 在 Toolkit 或驱动安装、升级及开机时生成 `/var/run/cdi/nvidia.yaml`。驱动卸载和 MIG 重配需要另行处理刷新。对 `1.18.2` 的节点，可用 `nvidia-ctk cdi list` 检查本地可列出的设备名，用 `journalctl -u nvidia-cdi-refresh.service` 看生成记录。再用 containerd 的有效目录配置与创建请求核对实际消费者。 [Toolkit 1.18.2 CDI 生成与刷新](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.18.2/cdi-support.html)

对照三种请求时，保持普通 runc，并假设无其他注入输入。省略 `CDI_devices` 时，即使节点有 B 的描述，基础 OCI 配置也不会应用 B 的修改。传正确全名且规格有效时，B 的节点和相关文件会进入修改后的配置。请求名拼错且在消费者缓存中找不到时，创建失败。若名字正确仍解析失败，核对消费者扫描的目录、规格解析报错及是否过期。若已找到名字但在应用或启动时失败，继续核对规格引用的宿主机路径与设备。 [CDI 名称解析与应用](https://github.com/cncf-tags/container-device-interface/blob/v1.1.0/pkg/cdi/cache.go) · [containerd 2.3.0 CRI 实现](https://github.com/containerd/containerd/blob/v2.3.0/internal/cri/server/container_create_linux.go)

## 三条路径各由谁修改容器

| 路径 | 本例的输入 | 修改者与时机 | 优先核对的证据 |
| --- | --- | --- | --- |
| containerd 原生 CDI | CRI `CDI_devices` 中的设备全名和节点上的规格 | containerd CRI 在组装 OCI 配置时应用 CDI 修改。 | 请求中的全名、containerd 扫描目录、创建结果和组装后的 OCI 配置。 |
| NVIDIA Runtime 即时 CDI（JIT CDI） | 选中 NVIDIA 包装层，`mode=auto`，在 NVML 平台用普通 GPU UUID 或索引选择 | 包装层在低层 `create` 前按请求在内存生成 CDI 描述并应用修改。 | 本次 handler、包装层解析出的模式、修改后的 OCI 配置。 |
| NVIDIA Runtime 显式 legacy | 选中 NVIDIA 包装层，`mode=legacy`，以 `NVIDIA_VISIBLE_DEVICES` 和 `NVIDIA_DRIVER_CAPABILITIES` 提供输入 | 包装层加入 `prestart` Hook，runc 创建时由 Hook 调用 CLI 配置容器。 | 模式、Hook 与 CLI 日志、容器里实际生效的设备和挂载。 |

NVIDIA Runtime 自 `1.18.0` 起将本例 NVML 平台的默认路径改为即时 CDI。在 `1.18.2` 中，若传给包装层的是全限定 CDI 名，`auto` 可解析为 `cdi`，由包装层查找持久规格。此处的调用者是包装层，原生 CDI 的调用者是 containerd CRI。分开采集三条路径，尤其要检查已有的 legacy Hook 是否也在相同容器上被触发。 [Toolkit 1.18.0 发行说明](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.18.0/release-notes.html) · [Toolkit 1.18.2 模式选择](https://github.com/NVIDIA/nvidia-container-toolkit/blob/v1.18.2/internal/info/auto.go) · [Toolkit 1.18.2 CDI 修改实现](https://github.com/NVIDIA/nvidia-container-toolkit/blob/v1.18.2/internal/modifier/cdi.go) · [Toolkit 1.18.2 CDI 与 legacy Hook 并存说明](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.18.2/cdi-support.html#running-a-workload-with-cdi)
