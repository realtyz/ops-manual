# PCIe 插卡、SXM 模组与 HGX GPU 底板

采购报价单上写着一行“HGX H100 8-GPU”，或者同样标着“H100 80 GB”的两款设备，价格、功耗和机箱厚度却差出一大截——只凭这一行字，能确定机柜该留几 U、配电该拉多大、多张卡之间怎么通信吗？

完全不够。在 GPU 基础设施里，**GPU 芯片、物理安装形态、多卡互联底板和服务器整机**是四个层层嵌套的边界。拆不清这四层，轻则机柜装不下、电力超标，重则买回来的服务器卡间通信跑不出预期带宽。

## PCIe 插卡 vs SXM 模组：形态背后是功耗与互联的天花板

GPU 放进服务器，最常见的有两种物理安装形态：**PCIe 插卡**与 **SXM 模组**。

**PCIe 插卡**就像传统服务器里的网卡或 RAID 卡，通过金手指插入主板或转接板的 PCIe 插槽。它胜在灵活、适配通用服务器，但也受限于插槽物理间距（常见双槽宽）、辅助供电线和风冷散热空间，单卡功耗上限往往被锁在三四百瓦左右。

**SXM 模组**则彻底抛弃了 PCIe 金手指，是 NVIDIA 专为高密算力设计的扣板封装形态。它直接贴装在专用的 **HGX GPU 底板**上，由底板统一提供高电流供电、超高速布线，并配合整机高风量风冷或液冷散热，把单卡功耗天花板推高到 700 W 级。

以 **HGX H100 8-GPU 底板**为例，8 颗 H100 SXM 模组与多颗 **NVSwitch** 交换芯片被共同封装在一块底板上，构成了整台服务器的“核心算力发动机舱”。此时，服务器内部清晰地分出两条独立的数据通道：

1. **主机 ↔ GPU（PCIe 通路）**：从主机内存向目标 GPU 搬运输入数据时，依然沿着前两篇讲过的 PCIe 桥树进入 GPU 子系统。
2. **GPU ↔ GPU（NVLink + NVSwitch 专网）**：当某张 GPU 需要把显存里的中间结果直接交给同机另一张 GPU 时，数据不再绕道拥挤的 PCIe 上行链路，而是经由专用的 **NVLink** 高速通道进入 **NVSwitch** 交换网络，再直达目标卡（具体路由、互联就绪服务与实测方法将在 [第 04 章](../04_单机多卡与GPU数据路径/00_内容大纲.md) 深入展开）。

![左侧为装入主板插槽的 PCIe 插卡。右侧将八个 SXM 模组及 NVSwitch 画在 HGX H100 底板内，CPU、主机内存、NIC、NVMe、BMC 和 PSU 位于整机范围。](images/fig-01-03-01-server-boundaries.png)

*图 01-03-01：PCIe 插卡形态（左）与基于 HGX H100 8-GPU 底板的服务器整机边界（右）。灰线将主机经 PCIe 接入 GPU 子系统的通路简化为单箭头；绿线表示各 GPU 经 NVLink 接入合并示意的 NVSwitch 互联网络；整机侧列出 CPU、主机内存、NIC、NVMe、BMC（基板管理控制器）与 PSU（电源模块）等外围子系统。*

## 同样叫 H100，为什么 SXM、PCIe 与 NVL 不能直接互换？

在硬件评审中最容易踩的坑，就是只盯着芯片名称和显存容量，误以为“都是 H100 80 GB，换个便宜版本就行”。下面对照 NVIDIA 官方数据表中的三款同代产品，看看封装形态改变究竟撬动了哪些硬指标：

| 关键规格 | H100 SXM | H100 PCIe 80 GB | H100 NVL |
| --- | --- | --- | --- |
| **物理形态** | SXM 模组，配专用 HGX 底板 | 双槽宽 PCIe 插卡，风冷 | PCIe 插卡，面向双卡 NVLink 桥接 |
| **单卡标称显存** | 80 GB | 80 GB | 94 GB |
| **最大热设计功耗（TDP）** | 最高 700 W（可配置） | 300–350 W（可配置） | 350–400 W（可配置） |
| **数据表列出的互联** | NVLink（接 NVSwitch）、PCIe Gen5 | NVLink（须配桥接件）、PCIe Gen5 | 双卡 NVLink 桥接、PCIe Gen5 |

表中的 **TDP（Thermal Design Power，热设计功耗）** 是部件级的功耗与散热设计标尺，用来指导整机厂商规划供电模组和风道散热能力，而不是卡在任意时刻的瞬时功耗。

> [!warning] 常见误区：标称显存相同，绝不等于可以直接替代
> 把 H100 SXM 换成 H100 PCIe 80 GB，单卡显存纸面上仍是 80 GB，但单卡功耗上限直接砍半（700 W → 350 W），物理插槽、辅助供电、散热风道以及多卡间能否全互联也彻底变了。如果报价单写的是 **H100 NVL**，单卡显存变成 94 GB、TDP 升至 350–400 W 且侧重双卡桥接形态，更必须严格按该型号的独立规格书核算，不能拿普通 PCIe 80 GB 套用。

> [!tip] 工程直觉：训练与推理的典型形态倾向
> - **8 卡 SXM / HGX 底板**：凭借高功耗释放与 8 卡全互联 NVSwitch，是大规模分布式训练和超大模型多卡张量并行（TP）推理的主力底座。
> - **PCIe 插卡 / H100 NVL**：更容易装进标准通用机架服务器，且 H100 NVL 凭借更高的单卡显存容量（94 GB）与双卡互联能力，常见于中小规模或弹性部署的大模型推理节点。

## 从 HGX 底板到服务器整机，再到机架：三级边界怎么拆？

读懂了卡型和底板，再往外看一层：**HGX H100 只定义了 8 张 GPU 及其互联底板，并不决定整台服务器长什么样。**

同一块 HGX H100 底板，交由不同整机厂商集成，会造出外形与外围配置完全不同的服务器：
- **NVIDIA DGX H100**：整机高度 **8U**，原生配备 8 张高速集群网卡、双口存储/管理网卡和统一参考架构。
- **Dell PowerEdge XE9680**（支持 HGX H100 SXM5 配置）：整机高度 **6U**，单台在垂直方向上比 DGX H100 少占 2U。

这里的 **U** 是标准机架的高度单位（1U 约 44.45 mm）。从手册里确实能直接算出 XE9680 单台少占 2U，但**机箱更矮不代表同一个机柜一定能塞进更多台服务器**——在真正上架前，还必须连同机箱深度、滑轨规格、机柜承重、供电回路容量、前后风道与背部线缆弯折空间一起过关。

为了避免采购或交付评审漏项，可以直接按下面这张“三级边界核对清单”逐层向厂商索要资料：

| 评审层级 | 必须拿到哪些规格资料 | 核心用来回答什么问题 |
| --- | --- | --- |
| **第一层：GPU 与安装形态** | 精确卡型（SXM / PCIe / NVL）、单卡显存、TDP 与互联桥接件 | 单卡显存够不够装模型？卡间互联有没有物理硬件支撑？ |
| **第二层：底板与整机拓扑** | HGX 底板型号、整机 PCIe/NUMA 拓扑框图、CPU/内存/NVMe/网卡数量与插槽 | 每张 GPU 与高速网卡是否对齐同一 NUMA？是否存在共享上行瓶颈？ |
| **第三层：整机与机架接口** | 机箱 U 数/深度/重量、导轨、进风方向与温度范围、交流输入功率与 PSU 冗余、BMC 口 | 机柜装不装得下？供电失一路会不会宕机？关机后怎么远程维护？ |

> [!summary] 核心结论
> - **分清四层边界**：芯片型号决定架构代际，安装形态（PCIe / SXM）划定功耗与散热上限，HGX 底板提供 8 卡 NVLink/NVSwitch 专网，整机型号最终敲定外围 I/O 与机架物理条件。
> - **看懂“同名不同命”**：SXM、PCIe 80 GB 与 NVL 在功耗、容量和互联形态上差异巨大，替换型号必须重查全部三层规格。

拿齐了第三层的整机接线与供电规格后，一台 10 kW 级的算力怪兽终于准备推进机房机柜。但在系统还没装好、甚至 Linux 卡死失联时，运维人员靠什么接管这台机器？双路配电万一断掉一路，服务器还能不能撑住？下一篇我们来看带外管理与机架运行的硬条件。

> [!info]- 参考资料
> - **HGX 底板与整机拓扑**：[NVIDIA HGX H100/H200 · 8-GPU 底板组件](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory-h100-h200-b200/latest/components.html#nvidia-hgx-h100-h200-8-gpu-baseboard)、[NVIDIA DGX H100/H200 · 系统拓扑](https://docs.nvidia.com/dgx/dgxh100-user-guide/introduction-to-dgxh100.html#dgx-h100-200-system-topology)
> - **H100 形态与型号规格**：[NVIDIA H100 数据表（SXM / PCIe）](https://i.dell.com/sites/csdocuments/App-Merchandizing_Documents/en/us/h100-gpu.pdf)、[NVIDIA H100 / H100 NVL 产品规格页](https://www.nvidia.com/en-us/data-center/h100/)
> - **整机机械与配置对比**：[NVIDIA DGX H100/H200 · 机械规格](https://docs.nvidia.com/dgx/dgxh100-user-guide/introduction-to-dgxh100.html#mechanical-specifications)、[Dell PowerEdge XE9680 · 系统配置](https://www.dell.com/support/manuals/en-us/poweredge-xe9680/xe9680_ism_pub/dell-poweredge-xe9680-system-configurations-and-features?guid=guid-219a12b6-5232-49fe-9506-ac2aed25ac30&lang=en-us)
