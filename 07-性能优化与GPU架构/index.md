# 07. 性能优化与GPU架构

> 考察候选人对GPU硬件架构的理解和实际性能调优能力

---

## 知识点

| 编号 | 知识点 | 关联题目数 |
|------|--------|-----------|
| 07.01 | 渲染提交优化（Draw Call/Batching/Instancing） | 3 |
| 07.02 | 剔除技术（视锥体/遮挡/距离） | 1 |
| 07.03 | GPU性能瓶颈分析与定位 | 2 |
| 07.04 | GPU执行模型（SIMT/Wave/Warp） | 1 |
| 07.05 | GPU内存层次与带宽优化 | 2 |
| 07.06 | Overdraw分析与优化 | 1 |
| 07.07 | GPU同步机制（Barrier/Sync） | 1 |
| 07.08 | GPU性能分析工具与流程 | 1 |
---

## 题目列表

### 初级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q07.01 | Draw Call是什么？为什么Draw Call过多会导致CPU瓶颈？常见的减少Draw Call的方法有哪些？ | 渲染提交优化（Draw Call/Batching/Instancing） | [查看](./draw-call-cpu-bottleneck-reduction.md) |
| Q07.02 | 静态批处理（Static Batching）和动态批处理（Dynamic Batching）的区别是什么？它们各自有什么限制？ | 渲染提交优化（Draw Call/Batching/Instancing） | [查看](./static-batching-vs-dynamic-batching.md) |

### 中级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q07.03 | GPU Instancing（实例化渲染）的原理是什么？在什么场景下使用Instancing最有效？ | 渲染提交优化（Draw Call/Batching/Instancing） | [查看](./gpu-instancing-principle-use-cases.md) |
| Q07.04 | 什么是Occlusion Culling（遮挡剔除）？请比较软件遮挡剔除和硬件遮挡查询（Occlusion Query）的优缺点。 | 剔除技术（视锥体/遮挡/距离） | [查看](./occlusion-culling-software-vs-hardware.md) |
| Q07.08 | 请解释Overdraw的概念。如何量化Overdraw？在高Overdraw场景下，延迟渲染和前向渲染哪个更优？为什么？ | Overdraw分析与优化 | [查看](./overdraw-analysis-deferred-vs-forward.md) |

### 高级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q07.05 | GPU的性能瓶颈通常分为哪几类（带宽瓶颈、计算瓶颈、顶点瓶颈、片元瓶颈）？如何通过实验判断当前是哪类瓶颈？ | GPU性能瓶颈分析与定位 | [查看](./gpu-bottleneck-classification-vertex-fragment-bandwidth.md) |
| Q07.06 | 什么是GPU Wave/Warp？SIMT（单指令多线程）执行模型的特点是什么？Warp Divergence如何影响性能？ | GPU执行模型（SIMT/Wave/Warp） | [查看](./gpu-wave-warp-simt-divergence.md) |
| Q07.07 | GPU的内存层次结构是怎样的？Shared Memory、L1 Cache、L2 Cache、VRAM的访问延迟和带宽分别是多少量级？ | GPU内存层次与带宽优化 | [查看](./gpu-memory-hierarchy-shared-memory-l2-vram.md) |

### 专家级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q07.09 | 什么是GPU Barrier/Sync？在多Pass渲染中，什么时候需要显式插入Barrier？错误的同步会导致什么问题？ | GPU同步机制（Barrier/Sync） | [查看](./gpu-barrier-sync-multipass-rendering.md) |
| Q07.10 | 请描述一次完整的GPU性能分析流程。你会使用哪些工具？重点关注哪些指标？ | GPU性能分析工具与流程 | [查看](./gpu-profiling-workflow-renderdoc-nsight-pix.md) |
| Q07.11 | 在移动端（Mali/Adreno/Apple GPU）上，渲染性能优化的策略与PC端有哪些关键差异？请讨论TBDR架构下的带宽优化策略。 | GPU内存层次与带宽优化, GPU性能瓶颈分析与定位 | [查看](./mobile-gpu-rendering-optimization-tbdr.md) |

