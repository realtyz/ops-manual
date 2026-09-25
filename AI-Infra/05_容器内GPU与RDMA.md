---
tags:
  - AI-Infra
  - GPU
  - RDMA
created: 2026-09-15
---

# 容器内 GPU 与 RDMA

> 本笔记对应 [[AI-Infra/00_简介|AI-Infra 大纲]] 的「阶段 5」。目标：把设备与数据面正确带进容器，并在容器内跑通 nccl-tests。
>
> **状态**：待展开（骨架）。

> [!todo] 待展开
> 按 [[AI-Infra/00_简介|大纲]] 阶段 5 的 checklist 逐条动手，把命令、输出、踩坑与结论写进这里；写完后回到大纲勾选。
>
> 边界提醒：本篇只讲**设备与数据面怎么进容器**；Device Plugin、GPU Operator、DRA、调度与配额属于编排层，见 `Kubernetes/` 的 GPU / AI 编排章。

> 关联：[[01_驱动与内核模块]]、[[03_RDMA与RoCE无损网络]]、`Kubernetes/` 的 GPU / AI 编排章
