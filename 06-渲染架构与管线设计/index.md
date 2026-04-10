# 06. 渲染架构与管线设计

> 考察候选人对不同渲染路径的深入理解及架构设计能力

---

## 知识点

| 编号 | 知识点 | 关联题目数 |
|------|--------|-----------|
| 06.01 | Forward与Deferred渲染对比 | 1 |
| 06.02 | G-Buffer设计与优化 | 1 |
| 06.03 | 半透明物体渲染方案 | 1 |
| 06.04 | Forward变体（Forward+/Clustered） | 2 |
| 06.05 | GPU Driven Rendering | 1 |
| 06.06 | 大世界与多线程渲染架构 | 2 |
| 06.09 | Render Graph 架构设计 | 1 |
| 06.10 | 透明物体渲染技术全景 | 1 |
---

## 题目列表

### 中级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q06.01 | 前向渲染（Forward Rendering）和延迟渲染（Deferred Rendering）的核心区别是什么？各自的优势和劣势是什么？ | Forward与Deferred渲染对比 | [查看](./forward-vs-deferred-rendering.md) |
| Q06.02 | 延迟渲染中G-Buffer通常包含哪些内容？G-Buffer的带宽开销如何估算？ | G-Buffer设计与优化 | [查看](./gbuffer-design-bandwidth-estimation.md) |
| Q06.03 | 延迟渲染为什么不能很好地处理半透明物体？常见的解决方案有哪些？ | 半透明物体渲染方案 | [查看](./deferred-transparency-oit-depth-peeling.md) |
| Q06.10 | 透明物体是怎么渲染的？ | 透明物体渲染技术全景 | [查看](./transparent-object-rendering-techniques.md) |

### 高级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q06.04 | Forward+（Forward Plus / Tiled Forward）渲染的原理是什么？它如何结合前向渲染和延迟渲染的优点？ | Forward变体（Forward+/Clustered） | [查看](./forward-plus-tiled-forward-light-culling.md) |
| Q06.05 | 什么是GPU Driven Rendering？请描述一个基于Compute Shader的GPU Driven管线的基本流程。 | GPU Driven Rendering | [查看](./gpu-driven-rendering-compute-shader.md) |
| Q06.06 | Clustered Rendering相比Tiled Rendering有什么改进？它如何处理光源在不同深度范围的归属问题？ | Forward变体（Forward+/Clustered） | [查看](./clustered-rendering-vs-tiled-depth.md) |

### 专家级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q06.07 | 请设计一个支持大世界、动态天气、日夜循环的渲染架构。如何组织渲染Pass、管理资源生命周期？ | 大世界与多线程渲染架构 | [查看](./open-world-day-night-cycle-rendering.md) |
| Q06.08 | 在多线程渲染架构中，Render Thread和Worker Thread的职责如何划分？如何避免GPU Bubble？ | 大世界与多线程渲染架构 | [查看](./multithread-rendering-render-thread-worker-thread.md) |
| Q06.09 | 什么是 Render Graph？它解决了传统渲染管线的哪些痛点？ | Render Graph 架构设计 | [查看](./render-graph-design-principles.md) |

