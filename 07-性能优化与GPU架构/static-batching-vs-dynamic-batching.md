---
id: Q07.02
title: "静态批处理（Static Batching）和动态批处理（Dynamic Batching）的区别是什么？它们各自有什么限制？"
chapter: 7
chapter_name: "性能优化与GPU架构"
difficulty: beginner
knowledge_points:
  - id: "07.01"
    name: "渲染提交优化（Draw Call/Batching/Instancing）"
tags: ["static-batching", "dynamic-batching", "mesh-combine"]
---

# Q07.02 静态批处理（Static Batching）和动态批处理（Dynamic Batching）的区别是什么？它们各自有什么限制？

**难度：** 🟢 初级
**所属章节：** [性能优化与GPU架构](./index.md)

---

## 🎯 结论

- Static Batching在构建时将多个静态网格的顶点数据合并为一个大的VBO，运行时只需一个Draw Call渲染所有合并的物体；Dynamic Batching在运行时逐帧将满足条件的小网格在CPU端合并顶点后提交渲染。两者目标相同（减少Draw Call），但适用场景、内存开销和性能特征截然不同。

## 📐 原理解析

### Static Batching的工作原理

- 构建阶段：将标记为Static的多个Mesh的顶点数据（位置、法线、UV等）合并到一个大的顶点缓冲中，同时为每个原始Mesh生成对应的索引子范围
- 运行时：对每个合并后的网格组只需一次Draw Call，通过不同的索引范围绘制不同的原始Mesh
- 材质要求：只有使用相同材质（或材质的Shader变体兼容）的物体才能合并到同一个Draw Call
- 内存代价：每个原始Mesh的顶点数据在合并后的VBO中都保留一份完整副本，因此Static Batching会增加内存占用

### Dynamic Batching的工作原理

- 运行时逐帧处理：每帧CPU遍历所有标记为可动态批处理的物体，检查是否满足合并条件（相同材质、顶点数在限制内）
- CPU端顶点变换：将每个物体的顶点数据从模型空间变换到世界空间（因为合并后失去了各自的模型矩阵）
- 合并提交：将满足条件的物体的变换后顶点数据拼接到一个临时缓冲区，提交一个Draw Call
- 逐帧开销：Dynamic Batching的合并工作每帧都要在CPU端重新执行，这是其主要的性能开销来源


## 🛠 工程实践

### Static Batching的最佳实践

- 适用于不移动、不旋转、不缩放的静态场景物体（建筑、地形、静态道具等）
- 在Unity中通过Inspector面板勾选Static → Batching，或在代码中使用StaticBatchingUtility.Combine
- 注意内存预算：在移动端，Static Batching的内存增加可能成为问题，需要权衡Draw Call减少和内存增加
- 对于大型开放世界，考虑按区域（Chunk）进行Static Batching，而非全局合并

### Dynamic Batching的最佳实践

- 适用于频繁移动的小型物体（粒子、小型道具、UI元素等）
- Unity中默认开启，可在Player Settings中全局控制开关
- 确保候选物体的材质相同——不同材质的物体无法动态批处理
- 注意顶点数限制：Unity默认限制为300顶点，可在Player Settings中调整（但不建议设得过大）


## ⚠️ 踩坑经验

### Static Batching的内存陷阱

Static Batching的核心代价是内存增加。假设场景中有100棵使用同一网格的树，未Batching时只需存储1份网格数据（通过实例变换矩阵区分），但Static Batching后需要存储100份完整的顶点数据。在大型场景中，内存增加可能达到数倍。对于大量重复物体，GPU Instancing是更节省内存的选择。

### Dynamic Batching的隐藏限制

- 顶点数限制：Unity默认300顶点/物体，超过限制的物体不会被动态批处理
- 材质限制：只有使用完全相同材质的物体才能合并，Material Property Block会导致无法批处理
- 不支持多Pass Shader：使用多Pass的Shader（如某些双面渲染Shader）的物体不会被动态批处理
- CPU开销随物体数量线性增长：物体越多，CPU端的遍历和合并开销越大，可能形成新的CPU瓶颈

### Dynamic Batching的顶点变换精度问题

Dynamic Batching在CPU端将顶点从模型空间变换到世界空间时，使用的是float精度。对于距离原点很远的物体，float精度不足可能导致顶点抖动（Jittering）。这是大型开放世界场景中常见的问题。


## 🔮 延伸思考

### GPU Instancing在多数场景下是更好的选择

对于大量重复物体（如树木、草地、柱子等），GPU Instancing既减少了Draw Call，又不需要像Static Batching那样增加内存。每个实例只需存储一个变换矩阵（64字节）而非完整顶点数据。在Unity中，GPU Instancing和Static Batching可以共存——引擎会自动选择更优的方案。

### Mesh Shader时代的Batching新范式

Mesh Shader（DX12/Vulkan扩展）允许在GPU端直接生成和变换网格，绕过了传统的Vertex Fetch阶段。这意味着可以在GPU端实现类似Batching的效果——一个Mesh Shader Dispatch可以渲染大量不同的网格片段，无需CPU端合并顶点数据。这是未来渲染架构的重要演进方向。

---

[← 返回 性能优化与GPU架构 题目列表](./index.md) | [返回总纲](../00-总纲.md)
