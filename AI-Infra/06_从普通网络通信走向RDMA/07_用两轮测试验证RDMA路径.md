# 用两轮测试把判断落到证据上

管理地址能连、RDMA 端口显示 `ACTIVE`，离“数据已从 A 的显存直接写到 B 的显存”还有两步。先固定设备、方向和 RDMA 操作，用主机内存跑通。再保持网络和请求参数，把两端的测试缓冲区换成 GPU 显存，确认测试确实完成了显存注册和传输。下面用 `ib_write_bw` 观察单向 RDMA Write。 [linux-rdma/perftest 26.07.8 · 测试对象和测量口径](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)

## 固定两轮试验的共同条件

下表与本文的结果表均为**自拟教学材料**。场景是两台裸机 Linux 服务器，每台选一张 NVIDIA GPU 和一个 ConnectX 端口。假定测试网络为已部署子网管理器的原生 InfiniBand，两端测试端口为 `ACTIVE`，`active_mtu` 允许使用 `4096 B`。A 是客户端和 Write 发起方，B 是服务端和写入目标。表中编号及地址为示例，两台主机恰好使用相同的本地编号。文中的源码和参数按 perftest `26.07.8` 核对，具体主机与驱动组合需要现场确认。

| 要对齐的条件 | A | B |
| --- | --- | --- |
| 角色和控制地址 | 客户端，`192.0.2.10` | 服务端，`192.0.2.20` |
| RDMA 设备、端口与 NIC BDF | `mlx5_0/1`，`0000:83:00.0` | `mlx5_0/1`，`0000:83:00.0` |
| GPU 的 CUDA 序号及 BDF | `0`，`0000:82:00.0` | `0`，`0000:82:00.0` |
| 测试路径 | 两个测试端口之间走原生 IB，载荷从 A 发往 B | NIC 与 GPU 在各自主机内接到同一 PCIe Switch |
| 放置与缓冲区 | 两轮采用相同的本地 CPU 放置，主机轮记录源缓冲区的 NUMA 位置 | 两轮采用相同的本地 CPU 放置，主机轮记录目标缓冲区的 NUMA 位置 |

现场采集时，用 `rdma link show mlx5_0/1` 和 `ibv_devinfo -d mlx5_0 -i 1` 核对端口、链路类型和 MTU。把工具看到的 CUDA 序号对回实际 BDF，尤其在更改 `CUDA_VISIBLE_DEVICES` 后。将两端的命令、程序退出状态和完整输出一起保存，并附上 GPU 与 NIC 型号、BDF、运行内核、GPU 驱动与模块类型、NIC 驱动和运行固件、实际加载的用户态库、`ib_write_bw -V` 及构建信息。换用 RoCE 时，另外核对所选端口的 GID、对应网络和路径 MTU。 [iproute2 v6.12.0 · rdma-link](https://github.com/iproute2/iproute2/blob/v6.12.0/man/man8/rdma-link.8) · [rdma-core v57.0 · ibv_devinfo](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/man/ibv_devinfo.1) · [linux-rdma/perftest 26.07.8 · `-d`、`-i`、`-m`、`-x`](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)

## 第一轮用主机内存，第二轮只更换缓冲区

第一轮由 B 先监听，再在 A 运行客户端。命令中固定 `RC`、端口、QP MTU、QP 数量、发送深度、消息大小及迭代数。两端采用匹配的测试参数，各自的 RDMA 设备和端口按本地映射填入。示例使用该版本的 TCP 控制连接交换参数，`-p 18515` 指定控制连接端口。载荷仍经 `-d` 和 `-i` 指定的 RDMA 设备及端口传输。若改用 `-R`，工具会使用 RDMA CM 建立测试用 QP，采集结果时应注明。 [linux-rdma/perftest 26.07.8 · 参数说明](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README) · [控制连接源码](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_communication.c)

```bash
# B 先启动
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits

# A 再启动，最后一个参数是 B 的控制地址
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits 192.0.2.20
```

第二轮在两端都显式选择 GPU 显存和 DMA-BUF。先查两端的 `ib_write_bw --help` 是否列出对应选项。该版本 README 的 CUDA DMA-BUF 条目列出 CUDA Toolkit `11.7` 及以上和 NVIDIA `515` 及以上开放内核模块。还要有支持 DMA-BUF MR 的 rdma-core 与 provider，具体硬件、固件及内核组合按所用平台确认。工具所选的设备要用实际 CUDA 序号核对，示例两端的序号都是 `0`。 [linux-rdma/perftest 26.07.8 · GPUDirect Usage](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README) · [工具参数与能力检测](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_parameters.c)

```bash
# B 先启动，显式选用本机 CUDA 设备和 DMA-BUF
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits \
  --use_cuda=0 --use_cuda_dmabuf

# A 再启动，载荷仍从 A 写向 B
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits \
  --use_cuda=0 --use_cuda_dmabuf 192.0.2.20
```

把第二轮的命令与双端日志一起保存，核对选中的 CUDA 设备、DMA-BUF 导出与 MR 注册分支、传输结果。该源码会在完整输出模式下打印 `Calling ibv_reg_dmabuf_mr(...)` 类信息。这个信息表示到达调用点。确认注册成功还要看调用后是否继续进入传输，并最终取得有效结果。CUDA 分配或导出失败要从 CUDA 侧的报错辨认，RDMA 导入失败则结合注册调用和错误输出定位。 [linux-rdma/perftest 26.07.8 · CUDA 分配与导出](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/cuda_memory.c) · [MR 注册和报错](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_resources.c)

## 先判定是否传通，再比较带宽

下表是**自拟的成功场景**，环境与命令如上。字段整理自应采集的双端信息，时间和速率用题设构造。真实 perftest 输出使用 `#bytes`、`#iterations`、`BW average[Gb/sec]` 等表头。下表的时间列为复算假设。单向测试的带宽以客户端 A 的测量口径为准。 [linux-rdma/perftest 26.07.8 · 测量方法](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README) · [结果表头](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_parameters.h)

| 轮次 | 两端缓冲区与注册路径 | 设备、操作和方向 | 消息大小 × 次数 | 假设计时 | 平均有效载荷带宽 | 双端退出 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 主机内存，普通 MR | A 的 `mlx5_0/1` → B 的 `mlx5_0/1`，RC Write | `65536 B × 8192` | `50 ms` | `85.90 Gbit/s` | `0 / 0` |
| 2 | 两端 CUDA `0` 的设备显存，DMA-BUF MR | 同上 | `65536 B × 8192` | `60 ms` | `71.58 Gbit/s` | `0 / 0` |

用第一行复算，`65536 B × 8192 = 536870912 B = 512 MiB`。假设这段净载荷在 `0.050 s` 内完成，`536870912 × 8 ÷ 0.050 ÷ 10^9 ≈ 85.90 Gbit/s`。第二行把分母换为 `0.060 s`，得到 `71.58 Gbit/s`。这里 `MiB` 用 `2^20 B`，`Gbit` 用 `10^9 bit`。单位换算用的时间是假设值，真实工具的迭代模式用客户端提交与完成时间戳计算平均带宽。 [linux-rdma/perftest 26.07.8 · 参数与带宽计算](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_parameters.c)

主机轮成功，为所选端口、方向和操作提供了主机内存 RDMA 的证据。第二轮若能把双端的显存分配、DMA-BUF MR 注册和传输成功对齐，就为所测的显存直达路径提供了证据。反向性能需要换向观测。数据内容正确性需要另外做校验，集合通信效率还取决于通信算法、并发和实际 GPU 放置。 [linux-rdma/perftest 26.07.8 · 测试方法及 Data Validation](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)

## 三种结果各需要哪一项补证

以下为**彼此独立的教学变体**。引用的短报错字符串来自指定版本源码，表中对场景的描述为自拟，实际判断应使用两端完整输出及退出状态。 [linux-rdma/perftest 26.07.8 · Write 测试流程](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/write_bw.c) · [MR 注册和错误处理](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_resources.c)

| 已确认的事件                                                                                            | 接下来查什么                                                                                                                                           |
| ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 主机轮未完成。若报 `Unable to init the socket connection`，控制连接阶段已经失败。若控制连接已建立，则看失败是否发生在本地资源创建、QP 配置或传输完成处。 | 控制失败时核对 B 是否监听 `18515` 及 A 到该地址的连通性。进入 RDMA 阶段后，核对两端选中的设备、端口和 QP 参数。本地打开或注册失败时再查 `uverbs` 权限、provider 和锁页限额。                                     |
| 主机轮成功。显存轮打印了 `Calling ibv_reg_dmabuf_mr(...)`，随后报 `Couldn't allocate MR` 一类错误。                    | 先确定失败端，保留完整报错和有记录的 `errno`。如果 CUDA 导出未成功，查分配类型、设备能力和页对齐。如果已进入 MR 导入调用，查 `fd`、范围、provider、NIC 驱动及固件，并核对 PCIe、ACS 和 IOMMU 条件。                      |
| 两轮都成功，显存轮较慢。                                                                                      | 先核对方向、消息大小、MTU、队列深度、NUMA 放置和统计口径。当前 A 的 NIC 读 A 显存，B 的 NIC 写 B 显存。对调发起方会对调两台机器上的读写角色，结合两端的 PCIe 路径可继续缩小范围。一次换向同时改变两端的角色，需额外控制一端或分别采集读写证据，才能继续归因。 |

排查记录可以按“控制连接 → RDMA 资源与请求 → GPU 缓冲区导出及注册 → 显存传输完成”逐项对齐。端口状态、主机轮和显存轮分别回答不同的问题，下一次检查从证据首次断开的地方开始。
