
`perftest` 是一组用来测量 RDMA 操作时延和带宽的微基准工具。本篇用其中的 `ib_write_bw` 验证一条单向 Write 路径。第一轮让两张网卡访问主机内存，先建立相同端口和方向上的 RDMA 基线。第二轮把两端缓冲区一起换成显存，观察 CUDA 导出、网卡导入和请求执行是否都能完成。这个对照覆盖两端网卡和指定缓冲区。要衡量训练中的 GPU 数据交换，还需考虑数据生产、消费、其他 Rank、通信算法和并发。 [perftest 26.07.8 的工具说明](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)

## 在同一对端口和 GPU 上固定比较条件

以下均为教学设定。两台裸机的测试端口走原生 **InfiniBand（IB）**，本例将它作为承载两端 RDMA 流量的网络。假定用于配置该网络的子网管理器已经运行，两端端口均为 `ACTIVE`，路径支持本例选择的 `4096 B` QP MTU。两台机器用了相同的本地编号，各编号只在所属机器和进程的运行环境中有意义。 [NVIDIA RDMA 编程手册 v1.7 的 InfiniBand 与端口概念](https://docs.nvidia.com/rdma-aware-networks-programming-user-manual-1-7.pdf) · [perftest 26.07.8 的端口和 MTU 参数](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)

| 条件 | 节点 A | 节点 B |
| --- | --- | --- |
| 角色及控制连接地址 | 客户端、Write 发起方，示例 IP `192.0.2.10` | 服务端、Write 目标，示例 IP `192.0.2.20` |
| RDMA 设备与端口 | `mlx5_0/1`，PCI `0000:83:00.0` | `mlx5_0/1`，PCI `0000:83:00.0` |
| CUDA 设备 | `0`，PCI `0000:82:00.0` | `0`，PCI `0000:82:00.0` |

两端的 GPU 和网卡各自在同一个 PCIe Switch 下。两轮保持控制通道、RDMA 端口、方向、CPU 绑定和请求参数一致，主机内存轮另记源与目标缓冲区的 NUMA 落点。运行前用 `ibv_devinfo -d mlx5_0 -i 1` 核对端口状态、`link_layer` 和 `active_mtu`，用 BDF 核对程序实际选中的 CUDA 设备。若换成 RoCE，需按所用网络条件核对端口的 GID（Global Identifier，全局标识）、相关接口与路径 MTU。 [rdma-core v57.0 的 ibv_devinfo](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/man/ibv_devinfo.1) · [perftest 26.07.8 的 GID 与 MTU 选项](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)

## 先跑主机内存，再注册显存

命令按 perftest 发行标签 `26.07.8` 的源码编写。运行时用实际的设备、端口和 B 的控制地址替换示例值，本例两端使用同一版本及构建。B 先启动服务端，A 后启动客户端。命令固定一条可靠连接（RC）的 QP，发送队列深度为 `128`，每条请求 `65536 B`（`64 KiB`），单方向重复 `8192` 次。默认的 TCP 控制连接通过 `-p 18515` 交换参数，`-d` 和 `-i` 选择承载载荷的 RDMA 端口。若用了 `-R`，则由 RDMA CM 建立测试用 QP，需要在结果中注明。 [perftest 发行页](https://github.com/linux-rdma/perftest/releases/tag/26.07.8) · [perftest 26.07.8 参数](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README) · [控制连接实现](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_communication.c)

```bash
# 第一轮，先在 B 启动
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits

# 再在 A 启动，最后一个参数是 B 的控制连接地址
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits 192.0.2.20
```

第二轮用同一组参数，加上 `--use_cuda=0 --use_cuda_dmabuf`，要求两端各自用 CUDA 设备 0 分配显存并走 DMA-BUF 注册。先用 `ib_write_bw --help` 查看这两个选项是否编入工具。该发行标签的 README 对 CUDA DMA-BUF 路径列出 CUDA Toolkit 11.7 或更高、NVIDIA 515 或更高版本的开放内核模块。还需有相应的 RDMA provider 和内核驱动支持。支持范围随平台和运行版本变化。 [perftest 26.07.8 的 GPUDirect 用法](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README) · [perftest 26.07.8 的 CUDA 参数](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_parameters.c)

```bash
# 第二轮，先在 B 启动
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits \
  --use_cuda=0 --use_cuda_dmabuf

# 再在 A 启动，载荷方向仍为 A → B
ib_write_bw -p 18515 -d mlx5_0 -i 1 -c RC -m 4096 \
  -q 1 -t 128 -s 65536 -n 8192 --report_gbits \
  --use_cuda=0 --use_cuda_dmabuf 192.0.2.20
```

对齐两端的命令、运行结果和退出状态。完整输出中的 `Calling ibv_reg_dmabuf_mr(...)` 标记代码走到导入调用。结合其后是否有注册错误、是否进入传输、A 是否取得有效结果，才能判断进展。若 CUDA 分配或导出失败，错误会出现在更早的 CUDA 路径。采集时同时记下 GPU 与 NIC 型号、BDF、内核、驱动及模块类型、运行固件、实际加载的用户态库、工具的构建与 `-V` 输出。 [perftest 26.07.8 的显存分配和导出](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/cuda_memory.c) · [MR 注册和报错](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_resources.c)

## 按单向有效载荷复算结果

以下为**自拟的成功记录**，保留 `#bytes`、`#iterations` 和平均带宽对应的统计含义，时间是为复算给出的假设值。两轮都是 A 通过一个 QP 写入 B，按 A 的测量口径，计算单向成功完成的应用载荷。实际迭代模式使用工具记录的提交与完成时间戳。 [perftest 26.07.8 的带宽计算](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_parameters.c) · [结果表头](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_parameters.h)

| 两端使用的缓冲区 | 每条消息 × 次数 | 假设计时 | 平均载荷带宽 | A / B 退出状态 |
| --- | --- | --- | --- | --- |
| 主机内存 MR | `65536 B × 8192` | `50 ms` | `85.90 Gbit/s` | `0 / 0` |
| CUDA 设备 0 的显存，已确认双端 DMA-BUF MR | `65536 B × 8192` | `60 ms` | `71.58 Gbit/s` | `0 / 0` |

每轮共搬运 `65536 × 8192 = 536870912 B = 512 MiB`。平均速率等于这段载荷的比特数除以计时区间。第一轮 `536870912 × 8 ÷ 0.050 ÷ 10^9 ≈ 85.90 Gbit/s`。第二轮将时间换成 `0.060 s`，得 `71.58 Gbit/s`。这里 `1 MiB = 2^20 B`，`1 Gbit = 10^9 bit`。数值按应用载荷计算，省去链路头部、重传及其他流量，也没有把两个方向或两块网卡的带宽相加。

主机轮证明所选端口在该方向上执行了主机内存 Write。显存轮要把双端显存分配、所用 MR 和成功执行对齐，才能为这一对 GPU 与网卡的直达路径提供证据。表中的两轮都走 RDMA，测量范围从已就绪的源缓冲区到目标缓冲区。要比较训练中的直达与中转，还须另计 GPU 与主机之间的复制及重叠。数据内容正确性要有内容校验，反向或多 Rank 表现要有相应规模和路径的测量。 [perftest 26.07.8 的测量和 Data Validation 说明](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)

## 从第一处缺失证据决定下一次对照

| 独立的教学情形 | 下一步补查 |
| --- | --- |
| 主机轮失败，报 `Unable to init the socket connection`。 | 先查 B 是否在控制地址和端口监听，以及 A 能否访问。若控制已成功而命令仍失败，按两端输出定位设备打开、本地注册、QP 配置或传输完成。 |
| 主机轮成功。显存轮在 B 到达 `ibv_reg_dmabuf_mr()`，随后报告 MR 分配失败。 | B 的程序已进入导入分支。保留 B 的完整输出和错误码，核对实际的 `fd`、范围、provider、NIC 驱动和固件，以及该 GPU–NIC 对的 PCIe 和 DMA 映射条件。 |
| 两轮成功，显存轮较慢。 | 先复核端口、MTU、消息、队列深度、NUMA 和统计窗口。当前 A 的网卡读显存，B 的网卡写显存。交换发起方会同时对调两个端点的读写角色。若要继续归因，可另做单端显存、另一端主机内存的对照，按测试程序实际支持的模式分别记录。 |

上表的状态和速率是教学材料，示例错误字符串取自对应版本。各阶段分别使用控制连接、设备与注册和传输完成的证据。 [perftest 26.07.8 的 Write 流程](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/write_bw.c) · [MR 注册错误](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_resources.c)
