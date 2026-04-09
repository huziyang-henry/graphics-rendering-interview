---
id: Q01.08
title: "在多Pass渲染中，如何设计Pass之间的数据传递？请讨论Render Target、UAV、Compute Shader等方案。"
chapter: 1
chapter_name: "渲染管线基础"
difficulty: advanced
knowledge_points:
  - id: "01.07"
    name: "多Pass数据传递与同步"
tags: ["render-target", "uav", "compute-shader", "multipass"]
---

# Q01.08 在多Pass渲染中，如何设计Pass之间的数据传递？请讨论Render Target、UAV、Compute Shader等方案。

**难度：** 🔵 高级
**所属章节：** [渲染管线基础](./index.md)

---

## 🎯 结论

- Pass间数据传递主要有三种方案：Render Target（读写分离，适合传统渲染）、UAV（随机读写，适合通用计算）、Compute Shader（完全灵活的通用计算）。
- Render Target通过Frame Buffer Object绑定纹理，适合结构化的图像数据传递。
- UAV和Compute Shader提供了更灵活的数据访问模式，适合GPU Particle、模拟计算和后处理等场景。

## 📐 原理解析

### Render Target（渲染目标）

- 通过FBO（Frame Buffer Object）将渲染输出到纹理而非屏幕
- Multiple Render Target（MRT）：一次渲染同时输出到多个纹理（最多8个，取决于硬件），每个对应FS中的不同输出
- G-Buffer是MRT的典型应用：Position/Normal/Albedo/Metallic等分别写入不同RT
- 下一Pass通过将上一Pass的输出纹理绑定为采样器来读取数据
- 限制：只能从FS输出（结构化写入），下一Pass只能通过采样器读取（只读），无法随机写入

### UAV（Unordered Access View）

- 允许在Shader中随机读写缓冲区或纹理，不受渲染管线的结构限制
- 在DX12中称为UAV，在Vulkan中称为Storage Image/Storage Buffer
- 支持原子操作（Atomic Operations）：InterlockedAdd、InterlockedCompareExchange等
- 可以在VS/HS/DS/GS/PS/CS中使用（不同API支持范围不同）
- 典型应用：Order-Independent Transparency（OIT）、GPU粒子计数、密度体积更新

### Compute Shader

- 完全独立于图形管线的通用计算阶段，不依赖光栅化
- Dispatch模型：Thread Group（工作组）→ Thread（线程），支持Shared Memory（工作组内共享）
- 可以读写SSBO/UAV/Texture，数据访问模式完全灵活
- 典型应用：图像后处理（Bloom、TAA）、物理模拟（流体、布料）、GPU Culling、Image-based Lighting预计算


## 🛠 工程实践

- Deferred Rendering中G-Buffer的格式设计：通常使用RGBA16_FLOAT或RGBA8_UNORM。Position可以用Depth Buffer重建（节省一个RT），Normal需要至少RGB10_A2精度以避免带状伪影。
- Compute Shader实现图像后处理：将图像作为SRV（Shader Resource View）读取，处理后写入UAV。使用GroupSharedMemory（Shared Memory）加速邻域采样（如高斯模糊的分离两Pass优化）。
- GPU Culling：使用Compute Shader执行视锥体剔除和遮挡剔除，将可见物体的Draw Command写入Indirect Arguments Buffer，然后通过ExecuteIndirect/DrawMeshTasksIndirect执行。这是GPU Driven Rendering的核心技术。
- Async Compute：将独立的Compute Pass（如后处理）和Graphics Pass并行执行，利用GPU的空闲计算单元。需要仔细管理资源依赖（Barrier）以避免数据竞争。

## ⚠️ 踩坑经验

- RT格式选择对带宽的影响：使用RGBA16_FLOAT而非RGBA8_UNORM会使带宽翻倍。在移动端尤其需要注意，过高的RT格式会导致带宽瓶颈。
- UAV的竞争条件（Race Condition）：多个线程同时读写同一内存位置时，需要使用原子操作或Barrier同步。错误的同步会导致数据损坏和视觉瑕疵。
- Implicit vs Explicit Barrier：在Vulkan/DX12中，资源状态转换（Resource Barrier）必须显式声明。遗漏Barrier可能导致读取到未完成的写入结果（数据竞争），或过度使用Barrier导致性能下降。
- Render Target的读写冲突：同一Pass中不能同时将纹理绑定为RT和采样器（读写同一资源需要中间Pass或Copy）。DX12的Simultaneous-Access Textures可以部分缓解此限制。

## 🔮 延伸思考

- Async Compute如何利用Pass间的空闲时间？GPU的Graphics Pipeline和Compute Pipeline使用不同的硬件单元。当Graphics Pipeline在处理VS/GS时，Compute单元可能空闲。通过将独立的Compute任务（如后处理、粒子模拟）调度到这些空闲时段，可以提高GPU整体利用率。关键是正确识别任务间的依赖关系。
- Implicit Barrier（如Vulkan的subpass dependency）和Explicit Barrier（如vkCmdPipelineBarrier）的区别：Implicit Barrier由API在特定同步点自动插入，开销较低但粒度粗；Explicit Barrier由开发者手动插入，粒度细但需要深入了解硬件行为。
- 未来的渲染架构可能更加Compute-centric：随着Mesh Shader和Ray Tracing的普及，传统的VS→FS管线正在被更灵活的Compute-based管线补充甚至替代。理解Compute Shader的数据传递模式对于现代渲染工程师越来越重要。

---

[← 返回 渲染管线基础 题目列表](./index.md) | [返回总纲](../00-总纲.md)
