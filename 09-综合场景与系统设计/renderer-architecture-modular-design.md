---
id: Q09.05
title: "如果让你从零开始设计一个现代游戏引擎的渲染器，你会如何划分模块？请画出整体架构图并解释各模块的职责和交互方式。"
chapter: 9
chapter_name: "综合场景与系统设计"
difficulty: expert
knowledge_points:
  - id: "09.05"
    name: "渲染器模块化架构设计"
tags: ["renderer", "architecture", "frame-graph", "rhi", "scene-management"]
---

# Q09.05 如果让你从零开始设计一个现代游戏引擎的渲染器，你会如何划分模块？请画出整体架构图并解释各模块的职责和交互方式。

**难度：** 🟣 专家级
**所属章节：** [综合场景与系统设计](./index.md)

---

## 🎯 结论

- 现代渲染器采用分层模块化架构，从底向上分为：RHI层（硬件抽象）→ 资源层（纹理/网格/Shader管理）→ 场景层（场景图和剔除）→ 渲染器层（Pass组织和执行）→ 特性层（阴影/GI/后处理等）。
- Frame Graph作为渲染器层的核心，负责声明资源依赖、推断执行顺序、管理资源生命周期和自动插入Barrier。
- 模块间通过明确的接口通信，避免循环依赖。数据驱动的配置方式使得渲染策略可以在不修改代码的情况下调整。

## 📐 原理解析

### RHI层（Render Hardware Interface）

- 职责：封装GPU API差异，提供统一的设备管理、资源创建、命令录制和同步接口
- 关键接口：RHIDevice、RHICommandBuffer、RHIPipelineState、RHIBuffer、RHITexture、RHIShader
- 设计原则：零开销抽象（Zero-overhead Abstraction）、延迟执行（Deferred Execution）

### 资源管理层

- Texture Manager：纹理加载、压缩、流式加载、缓存管理
- Mesh Manager：网格加载、LOD管理、骨骼动画数据
- Shader Manager：Shader编译、变体管理、热重载
- Material System：材质定义、参数绑定、材质实例

### 场景管理层

- Scene Graph：空间组织（BVH/Octree）、层级变换
- Culling System：视锥体剔除、遮挡剔除、距离剔除
- Light Manager：光源收集、光照探针、阴影光源选择

### 渲染器层（Frame Graph）

- Frame Graph：声明式API——Pass注册资源读写关系，自动推断执行顺序和Barrier
- Render Pass组织：Shadow Pass → G-Buffer Pass → Lighting Pass → Translucent Pass → Post Process Pass
- 资源生命周期管理：Transient Resource（Pass间临时资源）自动创建和回收

### 特性层

- Shadow System：CSM、Spot Light Shadow、Point Light Shadow
- GI System：Lightmap Baking、Light Probe、实时GI
- Post Process：Bloom、TAA、Tone Mapping、DOF
- Particle System：GPU Particle、Sprite Renderer


## 🛠 工程实践

- Frame Graph的实现参考：UE的Render Dependency Graph（RDG）、Unity的Render Graph API、Frostbite的Frame Graph。核心思想是让Pass开发者只关心"我需要什么资源"，而不用关心"资源从哪来、何时创建/销毁"。
- 数据驱动的设计：渲染配置（分辨率、LOD距离、Shadow Map大小等）通过JSON/Lua脚本定义，美术和技术美术可以调整参数而无需修改C++代码。
- 多线程架构：Game Thread（逻辑）→ Render Thread（剔除+命令录制）→ RHI Thread（驱动提交）→ GPU执行。Worker Thread通过Job System并行处理剔除和资源加载。
- 调试和验证：Frame Graph的Visualizer（可视化工具）可以展示每帧的Pass执行图、资源依赖关系和各Pass耗时，是调试渲染管线的重要工具。

## ⚠️ 踩坑经验

- 模块边界的划分粒度：过粗导致模块间耦合（如场景管理和渲染器直接互相调用），过细导致接口调用开销和开发复杂度增加。经验法则是"每个模块应该可以独立替换"。
- 渲染器和游戏逻辑的线程边界：渲染线程需要读取游戏逻辑的数据（变换矩阵、动画状态等），需要设计线程安全的数据交换机制（双缓冲或Copy-on-Write）。
- 资源生命周期的管理复杂度：GPU资源的使用生命周期可能跨多帧（如Streaming Texture），需要引用计数或Frame Graph的自动管理来避免资源过早释放或泄漏。
- Frame Graph的Transient Resource策略：临时RT的复用可以减少显存分配开销，但需要正确推断资源的生命周期（从最后一次Write到最后一次Read）。

## 🔮 延伸思考

- ECS（Entity Component System）架构对渲染器的影响：ECS的数据布局（SoA - Structure of Arrays）天然适合GPU Instancing和Compute Shader处理。现代引擎（如Unity DOTS）正在将渲染数据迁移到ECS架构。
- Render Graph vs Scene Graph的关系：Scene Graph管理逻辑层面的空间关系，Render Graph管理GPU层面的执行依赖。两者通过Culling System连接——Scene Graph提供可见物体列表，Render Graph消费这些数据。
- 未来的渲染器架构趋势：更加Compute-centric（Compute Shader承担更多工作）、更加Data-driven（配置化渲染策略）、更加GPU Driven（GPU端完成剔除和排序）。

---

[← 返回 综合场景与系统设计 题目列表](./index.md) | [返回总纲](../00-总纲.md)
