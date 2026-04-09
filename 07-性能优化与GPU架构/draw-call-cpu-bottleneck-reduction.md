---
id: Q07.01
title: "Draw Call是什么？为什么Draw Call过多会导致CPU瓶颈？常见的减少Draw Call的方法有哪些？"
chapter: 7
chapter_name: "性能优化与GPU架构"
difficulty: beginner
knowledge_points:
  - id: "07.01"
    name: "渲染提交优化（Draw Call/Batching/Instancing）"
tags: ["draw-call", "cpu-bound", "batching", "instancing"]
---

# Q07.01 Draw Call是什么？为什么Draw Call过多会导致CPU瓶颈？常见的减少Draw Call的方法有哪些？

**难度：** 🟢 初级
**所属章节：** [性能优化与GPU架构](./index.md)

---

## 🎯 结论

- Draw Call是CPU向GPU提交渲染命令的调用，每次调用涉及状态验证、资源绑定和命令写入等CPU开销。当Draw Call数量过多时，CPU端的命令准备和提交成为渲染管线的瓶颈，导致GPU处于饥饿状态（Idle），无法充分发挥计算能力。

## 📐 原理解析

### Draw Call的CPU开销来源

- 渲染状态设置：每次Draw Call前，CPU需要设置Blend模式、Depth/Stencil状态、光栅化参数等，这些状态变更涉及驱动层的验证和校验
- 资源绑定：将顶点缓冲（VB）、索引缓冲（IB）、纹理（Texture）、Uniform Buffer（UBO）等资源绑定到渲染管线，驱动需要检查资源格式和兼容性
- 命令写入：CPU将渲染命令写入命令缓冲区（Command Buffer），每个Draw Call对应一次glDrawElements/drawIndexed等API调用
- 状态验证：驱动层需要验证当前状态组合是否合法（如某些状态组合在某些GPU上不被支持），这一验证过程在旧版API（OpenGL/D3D11）中尤为昂贵

### CPU瓶颈的形成机制

GPU的并行计算能力远超CPU的单线程命令提交能力。一个典型的Draw Call可能只需GPU几微秒执行，但CPU准备该Draw Call可能需要数十微秒。当Draw Call数量达到数千甚至数万时，CPU无法足够快地提交命令，GPU大量时间处于等待状态。这就是所谓的'CPU Bound'场景。


## 🛠 工程实践

### Static Batching（静态批处理）

在构建阶段（Build Time）将多个静态网格合并为一个大的顶点缓冲，多个物体共享同一个材质的Draw Call合并为一个。适用于不移动、不旋转、不缩放的静态场景物体。Unity中只需勾选Static标志即可自动处理。

### Dynamic Batching（动态批处理）

在运行时（Runtime）逐帧将满足条件的小网格合并为一个Draw Call。适用于粒子系统、小道具等动态物体。Unity中默认开启，但有顶点数限制（默认300顶点）和材质限制。

### GPU Instancing（实例化渲染）

对同一网格的多次渲染使用一次Draw Call，通过Per-Instance数据（如变换矩阵、颜色）区分各实例。适用于大量重复物体（树木、草地、建筑元素）。Unity中通过Material属性或Graphics.DrawMeshInstanced使用。

### SRP Batcher（Shader变体优化）

Unity的Scriptable Render Pipeline Batcher通过将Shader属性按类型分组存储，减少每次Draw Call时的状态切换开销。它不需要合并网格，而是优化了CPU端设置材质属性的速度，对使用大量不同材质的场景效果显著。

### Indirect Draw（间接绘制）

现代API（Vulkan/DX12/Metal）支持的间接绘制允许GPU端决定Draw Call的参数（通过存储在缓冲区中的Draw Arguments），CPU只需提交一个Indirect Draw Call即可触发大量实际绘制。这是GPU Driven Rendering的基础。


## ⚠️ 踩坑经验

### Dynamic Batching的CPU开销可能超过收益

Dynamic Batching需要在CPU端逐帧遍历所有候选物体、合并顶点数据、变换顶点位置。当候选物体数量很大或顶点数据较复杂时，CPU合并开销可能远超减少Draw Call带来的收益。建议通过Profiler对比开启前后的CPU耗时来决定是否启用。

### Instancing的Per-Instance数据限制

GPU Instancing的Per-Instance数据有上限（通常受限于Constant Buffer大小，Unity默认最多支持256个Per-Instance属性）。超出限制时需要拆分为多个Instanced Draw Call，或使用Structured Buffer存储更多实例数据。此外，不同材质变体（Material Variant）的物体无法合并到同一个Instanced Draw Call。

### Batching破坏Frustum Culling粒度

Static Batching将多个物体合并为一个网格后，Frustum Culling只能对整个合并后的网格进行剔除。如果合并的物体在空间上分布较广，可能导致部分物体在视锥体外但整个网格仍在渲染。解决方案是在Static Batching前合理规划物体的空间分组。


## 🔮 延伸思考

### Multi-Draw Indirect进一步减少Draw Call开销

Vulkan的vkCmdDrawMultiIndexedIndirectEXT和DX12的ExecuteIndirect允许一次API调用提交多个Indirect Draw，进一步减少CPU端的API调用开销。结合GPU Compute Shader生成Draw Arguments，可以实现完全的GPU Driven Rendering Pipeline。

### GPU Driven Rendering的目标

GPU Driven Rendering的终极目标是消除CPU端的Draw Call瓶颈——CPU只提交少数几个Indirect Draw Call，所有可见性判定、LOD选择、排序、实例数据生成都由GPU的Compute Shader完成。Frostbite、UE5的Nanite都在向这个方向演进。

---

[← 返回 性能优化与GPU架构 题目列表](./index.md) | [返回总纲](../00-总纲.md)
