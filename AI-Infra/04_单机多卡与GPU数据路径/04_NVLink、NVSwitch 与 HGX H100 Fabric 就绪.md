
**NVLink** 是 NVIDIA 的高速设备互联。支持直连的两张 GPU 可以经 NVLink 访问彼此的显存，分担原来需要经 PCIe 承载的卡间流量。多张 GPU 要互达时，**NVSwitch** 可以在不同 GPU 的 NVLink 端口之间转发事务。GPU、交换芯片和链路组成这里所说的 **NVLink Fabric**，即一套能让这些设备互通的交换结构。CUDA 仍以访问者和对端显存描述 P2P，Fabric 提供底层路径。以下单独使用 HGX H100 八卡裸机案例。 [NVIDIA · 第三代 NVSwitch](https://developer.nvidia.com/blog/upgrading-multi-gpu-interconnectivity-with-the-third-generation-nvidia-nvswitch/) · [CUDA 11.8 · Hopper 第四代 NVLink](https://docs.nvidia.com/cuda/archive/11.8.0/hopper-tuning-guide/index.html#fourth-generation-nvlink)

## HGX H100 的主机 PCIe 与卡间 NVLink 路径

底板有八张 H100 SXM 和四个第三代 NVSwitch。每张 GPU 用 18 条第四代 NVLink 连到四个交换芯片。在图中，GPU 1 到四个交换芯片依次有 4、5、5、4 条链路。若 GPU 2 读取 GPU 1 的显存，请求可沿 GPU 2 → NVSwitch → GPU 1 到达对端，返回载荷沿 GPU 1 → NVSwitch → GPU 2 进入读取方。两卡显存各有归属。 [NVIDIA Fabric Manager · HGX H100 底板拓扑与连接表](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html#the-nvidia-hgx-h100-gpu-baseboard)

![HGX H100 底板有八张 GPU 和四个 NVSwitch。灰色虚线画出 GPU 1 的主机 PCIe 接入，四束琥珀色 NVLink 各接一个交换芯片，依次为 4、5、5、4 条。](images/fig-04-04-01-hgx-h100-pcie-and-nvlink.png)

**图 04-04-01**　HGX H100 八卡底板的逻辑拓扑。只展开 GPU 1 的接线。其他 GPU 到 NVSwitch 的 NVLink、其余主机 PCIe 连接均从略。GPU 与 NVSwitch 都作为 PCIe 设备出现在主机的 PCIe 树中。图中 GPU 编号用于定位底板连接，运行时用 UUID 核对实际设备。 [NVIDIA Fabric Manager · HGX H100 底板](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html#the-nvidia-hgx-h100-gpu-baseboard)

第四代 NVLink 每条链路为 `50 GB/s` 双向合计。每张 H100 SXM 的 18 条链路合计 `18 × 50 = 900 GB/s`，按对称方向拆开是 `18 × 25 = 450 GB/s` 的单方向聚合规格。估算一次单向卡间复制时，要考虑源端发出、目的端接收、实际使用的链路和交换结构上的共享流量。主机向 H100 送入数据时，则沿主机 PCIe 路径估算，使用相应链路与内存系统的预算。 [NVIDIA · Hopper 第四代 NVLink](https://docs.nvidia.com/cuda/archive/11.8.0/hopper-tuning-guide/index.html#fourth-generation-nvlink)

## ALI 链路训练、Fabric Manager 与 GPU 注册

在 HGX H100 上，ALI（Autonomous Link Initialization，自主链路初始化）让 GPU 和 NVSwitch 硬件训练 NVLink，建立链路层通信。训练建立每个 GPU 端口与交换芯片之间的链路。要让事务经交换芯片到达另一张卡，主机上的 Fabric Manager（FM）与驱动协作配置路由并监视 Fabric，GPU 还需向 Fabric 注册才能获得 NVLink P2P 能力。按 NVIDIA 的 H100 裸机说明，某张卡注册失败会失去 NVLink P2P 能力，该 GPU 仍可供非 P2P 用途使用。 [NVIDIA Fabric Manager · 核心软件栈](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html#nvswitch-core-software-stack) · [初始化 NVSwitch 和 NVLink](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html#initializing-nvswitch-and-nvlink)

假设作业需要 GPU A 和 GPU B 做 NVLink 传输，A、B 各对应一个已经核对的 UUID。NVIDIA 的 `nvidia-smi -q` 单 GPU 示例用 `In Progress / N/A` 表示正在注册，用 `Completed / Success` 表示注册成功。以下把这些字段编入一轮教学观察，记录可见设备、服务与 GPU 的状态。 [NVIDIA Fabric Manager · Fabric 注册状态示例](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html#initializing-nvswitch-and-nvlink)

| 观察 | 当前能确认的状态 | 继续核对 |
| --- | --- | --- |
| PCIe 树列出设备，`nvidia-smi` 查到 A、B | 主机枚举及管理查询已通 | 采集两端 NVLink 链路及 Fabric 状态 |
| 链路已建立，FM 服务 `active`，A 的 Fabric 为 `In Progress / N/A` | 硬件链路已建立，查询时 FM 在运行，A 正在注册 | 对齐同一启动周期的 FM 日志和 GPU 注册变化 |
| A 为 `Completed / Success`，B 仍在注册 | A 已注册，B 的注册尚未结束 | 查 B 后续的 `State` 和 `Status` |
| 两端均注册成功，工具报告所需方向的 NVLink P2P 能力 | 具备对这组设备发起功能验证的条件 | 在目标设备对上执行指定方向的访问或传输，等待完成并校验数据 |

在使用 systemd 的裸机上，可用 `systemctl status nvidia-fabricmanager.service` 查询服务，`journalctl -b -u nvidia-fabricmanager.service -o short-iso` 对齐本次启动的日志。用 `nvidia-smi -q -i <GPU UUID>` 逐卡取 Fabric 字段，用 `nvidia-smi nvlink -s` 看链路状态，用 `nvidia-smi topo -p2p n` 取 NVLink P2P 矩阵。现场记录还应附工具、驱动和 FM 版本。一次通过的设备对测试覆盖本次使用的方向、消息和实现。 [NVIDIA Fabric Manager · systemd 服务](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html#managing-the-fabric-manager-service) · [NVIDIA SMI · NVLink 与 Topology](https://docs.nvidia.com/deploy/nvidia-smi/)

## 第四代 NVSwitch 的 NVLSM 与软件版本

HGX B100/B200/B300 八卡平台采用第四代 NVSwitch。NVLSM（NVLink Subnet Manager，NVLink 子网管理器）发现拓扑并计算、下发交换机转发表，FM 负责 GPU 侧路由与 NVLink 配置等，两者通过进程间通信协作。因此检查这类平台时，服务和配置证据还要覆盖 NVLSM。 [NVIDIA Fabric Manager · 第四代软件栈](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html#nvswitch-core-software-stack)

第四代平台的裸机安装说明以驱动分支 620 为界。620 之前按驱动分支安装配套组件，从 620 起 FM 及相关组件可独立于 GPU 驱动安装，FM 仍依赖驱动提供的 `libnvidia-fmdrv.so.1`。读对应平台的安装表和版本依赖，才能判断某一组软件包是否匹配。HGX H100 属于第三代 NVSwitch，指南仍要求该平台的驱动侧库版本与 FM 匹配。 [NVIDIA Fabric Manager · 第四代及后续平台安装与版本](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html#systems-using-fourth-generation-nvswitches-or-later) · [Fabric Manager Versioning](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html#fabric-manager-versioning)
