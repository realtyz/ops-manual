# 用 perftest 验证主机内存与显存传输

多机训练步时偏慢，或者 PD 分离推理的 KV Cache 迁移耗时飙升时，直接上 NCCL 或推理框架排查往往一头雾水——框架并发、计算流水、集合算法与底层网络混在一锅汤里。工程上最稳妥的做法，是先用底层微基准工具 **`perftest`** 在单对网卡和 GPU 上做一把干净的“对照尺”。

本篇使用 `perftest 26.07.8` 中的 `ib_write_bw`，固定同一对端口与单向 RDMA Write 路径，分两轮推进：
1. **主机内存轮**：验证两端网卡、物理链路、RC QP 建连与主机内存 RDMA 基线；
2. **显存直达轮**：保持一切参数不变，只把两端缓冲区换成 GPU 显存并启用 DMA-BUF 注册，精准验证 GPUDirect RDMA 能否走通以及有效带宽表现。

## 固定对照条件：消除一切无关变量

下面的对照部署在两台裸机（`node-1` 与 `node-2`）的原生 InfiniBand 高速端口上（子网管理器已就绪，端口状态为 `ACTIVE`，路径支持 `4096 B` MTU）。两轮测试严格锁死控制通道、RDMA 端口、传输方向、CPU 核绑定与请求参数，唯一变量是缓冲区所在介质：

| 对照维度 | `node-1`（客户端 / Write 发起端） | `node-2`（服务端 / Write 目标端） |
| --- | --- | --- |
| **控制通道 IP** | `192.0.2.10` | `192.0.2.20` |
| **RDMA 端口与 BDF** | `mlx5_0/1`（`0000:83:00.0`） | `mlx5_0/1`（`0000:83:00.0`） |
| **CUDA 设备与 BDF** | 设备 `0`（`0000:82:00.0`） | 设备 `0`（`0000:82:00.0`） |
| **本地拓扑距离** | `PIX`（同 PCIe Switch） | `PIX`（同 PCIe Switch） |

开始跑数前，先在两端用 `ibv_devinfo -d mlx5_0 -i 1` 复核端口 `PORT_ACTIVE`、`link_layer` 与 `active_mtu`，并按 BDF 确认进程选中的确实是目标 GPU（主机内存轮还应顺手记录缓冲区的 NUMA 亲和落点）。

如果底层网络是 RoCEv2 而非原生 InfiniBand，还需额外核对端口的 GID 索引、关联以太网接口与路径 MTU（具体配置详见 [第 07 章](../07_InfiniBand、RoCE与集群网络拓扑/00_内容大纲.md)）。

## 两轮实操命令与关键输出标记

测试使用单条 RC QP、发送队列深度 `128`、单条消息 `65536 B`（`64 KiB`）、单向重复 `8192` 次。默认经 TCP 端口 `18515` 交换 QP 参数（若改用 `-R` 通过 RDMA CM 建连，记录结果时务必注明）。执行顺序永远是**先启服务端，再启客户端**：

```bash
# ===== 第一轮：主机内存 RDMA 基线 =====
# 1) 在 node-2（服务端 / 目标端）启动监听
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits

# 2) 在 node-1（客户端 / 发起端）发起单向 Write，末尾指向服务端控制 IP
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits 192.0.2.20
```

主机内存轮顺利拿到退出码 `0` 后，原封不动保留全部网络参数，仅在两端追加 `--use_cuda=0 --use_cuda_dmabuf` 切换到 GPU 显存与 DMA-BUF 路径（执行前先跑 `ib_write_bw --help` 确认编译时已打开 CUDA 与 DMA-BUF 支持，并留存 `ib_write_bw -V` 版本号）：

```bash
# ===== 第二轮：双端显存 GPUDirect RDMA（DMA-BUF）=====
# 1) 在 node-2（服务端）启用 CUDA 0 显存与 DMA-BUF 注册
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits \
  --use_cuda=0 --use_cuda_dmabuf

# 2) 在 node-1（客户端）发起同方向显存直达 Write
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits \
  --use_cuda=0 --use_cuda_dmabuf 192.0.2.20
```

下面是第二轮显存测试在客户端（`node-1`）的一段典型输出节选，重点留意导入标记、连接摘要和统计表头：

```text
Initializing CUDA
cuda_memory_init: allocating GPU memory on device 0 (0000:82:00.0)
Calling ibv_reg_dmabuf_mr(pd=..., offset=0, len=..., iova=..., fd=...)  # ① 确认进入 DMA-BUF 导入分支
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 TX depth        : 128
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW average[Gb/sec]
 65536      8192           71.58                                       # ② 单向应用载荷平均带宽
---------------------------------------------------------------------------------------
```

保留完整日志至关重要：如果失败发生在 CUDA 分配或导出阶段，终端会直接抛出 CUDA 错误；如果打印出了 `Calling ibv_reg_dmabuf_mr(...)` 之后才报错，就说明 GPU 导出已经过关，真正卡住的是 RDMA 侧导入或硬件映射。归档测试记录时，一并记下 GPU/NIC 型号、BDF、内核版本、驱动及模块类型、运行固件和用户态库版本。

## 独立复算单向有效载荷带宽

`perftest` 报告的 `BW average[Gb/sec]` 严格按单向成功完成的应用载荷比特数除以实测计时窗口算出（`--report_gbits` 采用十进制 $1\text{ Gb} = 10^9\text{ bit}$，而消息体积按二进制 $1\text{ MiB} = 2^{20}\text{ B}$）。

在本轮固定参数下，单次测试搬运的总应用有效载荷为：

$$
V = 65\,536\text{ B} \times 8\,192 = 536\,870\,912\text{ B} = 512\text{ MiB}
$$

假设两轮记录到的提交至完成实测计时分别为 $50\text{ ms}$（主机内存轮）和 $60\text{ ms}$（显存直达轮），手动复算平均有效载荷带宽如下：

$$
\text{BW}_{\text{主机轮}} = \frac{536\,870\,912 \times 8\text{ bit}}{0.050\text{ s} \times 10^9} \approx 85.90\text{ Gb/s}
$$

$$
\text{BW}_{\text{显存轮}} = \frac{536\,870\,912 \times 8\text{ bit}}{0.060\text{ s} \times 10^9} \approx 71.58\text{ Gb/s}
$$

将两轮核心证据汇总对照，就能一眼看清两条通路的状态与差距：

| 测试轮次与缓冲区类型 | 单向总载荷（消息 × 次数） | 实测计时窗口 | 平均载荷带宽 | 两端退出码 |
| --- | --- | --- | --- | --- |
| **第一轮：主机内存 MR** | $512\text{ MiB}$（`65536 B × 8192`） | `50 ms` | **`85.90 Gb/s`** | `0 / 0` |
| **第二轮：双端 CUDA 0 显存（DMA-BUF MR）** | $512\text{ MiB}$（`65536 B × 8192`） | `60 ms` | **`71.58 Gb/s`** | `0 / 0` |

> [!warning] 常见误区：不要过度解读单次 `ib_write_bw` 跑分
> 1. **统计口径**：该带宽仅核算单向有效载荷（不含协议头与链路控制帧），且既未叠加双向，也未叠加多块网卡。
> 2. **测试范围**：两轮测试都从“已经就绪的缓冲区”开始发包，不包含真实训练/推理中的 GPU 计算、同步事件、数据内容校验（需另开校验选项）以及多 Rank 集合通信并发竞争；同时 `node-1 → node-2` 的成绩也不能自动代表反向路径表现。

> [!tip] 工程直觉：面向推理负载的互补基准
> `ib_write_bw` 非常适合模拟训练大梯度桶以及 PD 分离推理中大块 KV Cache 的持续吞吐。
>
> 若在评估**推理 TP 小消息延迟**或 **Decode 端拉取模式（Pull）KV 迁移**，可加测两把尺子：用 `ib_write_lat` 测小载荷（`64 B`–`4 KiB`）往返微秒时延，用 `ib_read_bw --use_cuda=0 --use_cuda_dmabuf` 验单边 Read 吞吐。

## 排障决策表：从第一处缺失证据决定下一步

两轮对照最大的威力在于“分段定界”——遇到异常时，不用盲猜参数，直接按下面三类现象锁定下一步补查动作：

| 观察到的测试现象 | 故障定界与下一步精准动作 |
| --- | --- |
| **主机轮即失败**，报错 `Unable to init the socket connection` | 卡在**带外 TCP 控制连接**阶段，甚至还没碰 RDMA 硬件。先查服务端是否在目标 IP 与端口 `18515` 监听、防火墙与路由是否放通；控制连上后若仍失败，再沿两端输出排查设备打开、本地注册或 QP 建连。 |
| **主机轮成功；显存轮失败**，在 `node-2` 打印 `Calling ibv_reg_dmabuf_mr(...)` 后报 MR 分配失败 | CUDA 导出已通过，**断点落在 `node-2` 的 DMA-BUF 导入阶段**。保留完整报错码，依次核对 `node-2` 的 GPU–NIC BDF 配对、`ACSCtl`、`iommu_group/type`、mlx5 驱动与网卡运行固件。 |
| **两轮均成功，但显存轮明显慢于主机轮** | 先复核端口、MTU、队列深度、NUMA 与统计窗口是否一致。记住本轮中**发起端网卡在做 PCIe 读显存，远端网卡在做 PCIe 写显存**——直接互换客户端/服务端会同时对调两端读写角色；要单独定位哪一端拖慢，应做“一端用显存、另一端用主机内存”的**单端显存对照测试**。 |

把点对点链路验实之后，全章的核心概念、驱动证据、PCIe 条件与基准方法就全部拼合完毕。接下来进入综合检验篇，用三组贴近真实生产环境的无实机推演题，检验你能否独立把整条 RDMA 与 GPUDirect RDMA 链路贯通起来。

> [!info]- 参考资料
> - **`perftest 26.07.8` 官方发行与参数手册**：[perftest 26.07.8 Release](https://github.com/linux-rdma/perftest/releases/tag/26.07.8)、[perftest 26.07.8 README（GPUDirect / MTU / Data Validation）](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)
> - **`perftest 26.07.8` 关键源码参考**：[`src/write_bw.c`](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/write_bw.c)、[`src/perftest_communication.c`](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_communication.c)、[`src/cuda_memory.c`](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/cuda_memory.c)、[`src/perftest_resources.c`](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_resources.c)、[`src/perftest_parameters.c`](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_parameters.c)、[`src/perftest_parameters.h`](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_parameters.h)
> - **设备端口与 InfiniBand 概念参考**：[rdma-core v57.0 `ibv_devinfo(1)`](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/man/ibv_devinfo.1)、[NVIDIA RDMA Aware Networks Programming User Manual v1.7](https://docs.nvidia.com/rdma-aware-networks-programming-user-manual-1-7.pdf)
