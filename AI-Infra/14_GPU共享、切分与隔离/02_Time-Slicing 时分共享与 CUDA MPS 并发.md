
A、B 各启动一个 CUDA 进程，并获得了同一张 GPU 的访问权。**CUDA context（上下文）** 保存进程在该设备上执行所需的状态，包括地址空间和内存分配。让两个上下文轮流执行，可以利用某个进程等待输入时的空档。如果二者都在持续提交很小的 Kernel，还可以考虑让它们的工作同时推进。 [NVIDIA MPS R595 架构](https://docs.nvidia.com/deploy/mps/595/architecture.html)

## Time-Slicing 的执行时序与驻留显存

Time-Slicing（时分共享）让多个进程的工作在同一设备上交错执行。应用仍通过 CUDA 提交 Kernel，驱动负责调度各上下文。Kubernetes 中的 NVIDIA Device Plugin 可以给多个容器授予同一设备的访问名额。名额交付完成后，容器里的进程也可以向这张卡提交工作。下表假定两份缓冲区都放得下，描述一次可能的执行顺序，片段长度按实际工作变化。 [NVIDIA MPS 的背景与执行模型](https://docs.nvidia.com/deploy/mps/595/architecture.html#background) · [GPU Operator v26.7 Time-Slicing 语义](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html#understanding-time-slicing-gpus)

| 观察对象 | 片段一 | 片段二 | 片段三 |
| --- | --- | --- | --- |
| GPU 正在执行的工作 | A | B | A |
| 驻留显存 | A 与 B 的缓冲区 | A 与 B 的缓冲区 | A 与 B 的缓冲区 |

若改用 16 GiB 教学卡，且 A 保留 12 GiB，B 在准备 GPU 工作时申请 8 GiB，就会遇到容量不足。即使 A 此时完全没有 GPU 工作，B 仍只有 4 GiB 可用。换到具体设备也一样。NVIDIA 的 A100-SXM4-40GB 枚举例给出 40,537 MiB。若两个落到同卡的进程各需同时驻留 22 GiB，则总量为 `2 × 22 × 1024 = 45,056 MiB`，比该示例容量多 4,519 MiB。设备访问可先成功，显存申请随后失败。需求是假设，容量取自官方 R450 环境示例。 [NVIDIA A100 显存枚举例](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/getting-started-with-mig.html#enable-mig-mode)

再改变一个条件。若 B 的容器原先只有一个活跃 GPU 进程，改为启动三个，设备插件仍只看到一个已分配的名额，驱动看到的活跃进程则从总计两个变为四个。这会改变 A 与 B 的竞争。名额数只用于准入，实际等待要根据提交时机、工作长度和同卡活跃进程测量。单个进程的分配失败可以只影响该应用。设备级故障的影响范围则可能覆盖同一卡的多个共享进程。 [NVIDIA Time-Slicing 的进程调度与故障边界](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html)

## CUDA MPS 的客户端与并发条件

CUDA MPS（Multi-Process Service，多进程服务）是 CUDA 的协作运行服务，适合多个进程持续提交小 Kernel、各自只用到部分执行资源的场景。以 A100 上属于同一 Linux 用户的两个 CUDA 进程为例，管理员在 GPU 节点运行 MPS control daemon。进程建立 CUDA 上下文时连接 daemon，由其启动或选择 MPS server。A100 采用 Volta 及更新架构的执行模型，server 协调客户端共享 GPU 调度资源，各客户端保有自己的 GPU 地址空间，并直接向 GPU 提交工作。 [NVIDIA MPS R595 架构](https://docs.nvidia.com/deploy/mps/595/architecture.html)

以两份各能驻留的小 Kernel 为例。时分场景中，B 的工作等到自己的执行时段。通过 MPS，若 A 提交后仍有 SM 线程、寄存器等资源可供 B 的线程块驻留，两个 Kernel 就有机会重叠。随着 A 的线程块增多，剩余资源可能容不下 B 的线程块，重叠机会随之减少。若两者已压满同一份显存带宽，增加重叠也可能主要增加争用。应用应以固定输入下的总吞吐和各自的延迟比较两种执行方式。 [NVIDIA MPS R595 适用负载](https://docs.nvidia.com/deploy/mps/595/when-to-use-mps.html#identifying-candidate-applications)

## MPS 执行上限、显存上限与故障范围

| 配置 | 控制的量 | 如何核对 |
| --- | --- | --- |
| Active thread percentage | 客户端可用的执行资源上限，受硬件粒度限制。不同客户端的 Kernel 可以使用同一个 SM。 | 查 MPS 的有效配置，必要时在客户端读取 `cudaDevAttrMultiProcessorCount`。 |
| Pinned device memory limit | 客户端通过 CUDA API 可分配的设备显存上限，包含相应的 CUDA 内部分配。 | 查对应设备和客户端生效的限制，再测显存分配返回值。 |

如果把线程比例设为 25%，客户端即使单跑也受该上限约束。比较两个都设 25% 的客户端时，只能从配置推出各自最多能动用的执行资源受到限制。线程实际在哪些 SM 上执行、等待多长时间、各完成多少工作，还取决于 Kernel 与同卡负载。上表讨论百分比式限制，具体接口和舍入按所用文档和 daemon 查验。 [NVIDIA MPS R595 线程与显存限制](https://docs.nvidia.com/deploy/mps/595/when-to-use-mps.html) · [NVIDIA MPS R595 管理接口](https://docs.nvidia.com/deploy/mps/595/appendix-tools-and-interface-reference.html)

同一卡上的 A、B 虽各有地址空间，致命 GPU 故障仍可能把两者都纳入受影响范围。R595 文档的 Volta 及以后模型中，相关客户端接到故障通知，server 进入 `FAULT`，待相关客户端退出后重建受影响设备的上下文，再接受新客户端。这是读取 server 状态与客户端错误时应跟踪的恢复过程。 [NVIDIA MPS R595 错误隔离与恢复](https://docs.nvidia.com/deploy/mps/595/when-to-use-mps.html#memory-protection-and-error-containment)

时分和 MPS 都让多个进程在同一物理边界内工作。某些负载即使在小 Kernel 能够并发时，仍要求各自有显存容量与独立的内存访问路径。这一要求由下一篇的硬件实例来表达。
