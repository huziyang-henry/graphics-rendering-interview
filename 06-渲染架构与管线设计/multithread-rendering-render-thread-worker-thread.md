---
id: Q06.08
title: "在多线程渲染架构中，Render Thread和Worker Thread的职责如何划分？如何避免GPU Bubble？"
chapter: 6
chapter_name: "渲染架构与管线设计"
difficulty: expert
knowledge_points:
  - id: "06.06"
    name: "大世界与多线程渲染架构"
tags: ["multithreading", "render-thread", "command-buffer", "frame-graph"]
---

# Q06.08 在多线程渲染架构中，Render Thread和Worker Thread的职责如何划分？如何避免GPU Bubble？

**难度：** 🟣 专家级
**所属章节：** [渲染架构与管线设计](./index.md)

---

## 🎯 结论

- Render Thread 负责将渲染命令提交到 GPU，Worker Thread 负责并行执行剔除、排序、命令录制等 CPU 端工作。两者通过任务系统（Job System）和同步原语（Fence、Semaphore）协调。
- 典型的多线程渲染架构：Game Thread（游戏逻辑）→ Render Thread（命令提交）→ RHI Thread（驱动交互）→ GPU 执行。每一层通过队列和异步机制解耦。
- GPU Bubble（GPU 空闲等待 CPU 提交命令）是多线程渲染的主要性能问题，通过命令缓冲区的多线程录制、异步计算和帧间并行等策略来避免。

## 📐 原理解析

### 多线程渲染架构的层次结构

- Game Thread：执行游戏逻辑（物理、AI、动画），每帧结束时收集渲染所需的数据（Transform、Material、Light 等），提交给 Render Thread。
- Render Thread：接收 Game Thread 的渲染数据，执行视锥体剔除、排序、渲染状态设置，将命令录制到 Command Buffer 中，提交给 RHI Thread。
- RHI Thread（Rendering Hardware Interface Thread）：与图形驱动交互，将 Command Buffer 提交给 GPU。在 Vulkan/DX12 中，RHI Thread 对应一个独立的线程，负责 Queue 提交和 Fence 等待。
- GPU：异步执行渲染命令，与 CPU 线程并行工作。GPU 的执行进度通过 Fence/Semaphore 通知 CPU。
- Worker Thread（Job System）：由 Render Thread 派发的并行任务，执行剔除、LOD 选择、命令录制等可并行化的工作。

### GPU Bubble 的成因

- GPU Bubble 是指 GPU 在一段时间内处于空闲状态，等待 CPU 提交新的命令。这通常发生在 CPU 端的命令录制或提交速度跟不上 GPU 的消费速度时。
- 成因 1：Render Thread 的单线程瓶颈。当场景复杂度增加时，Render Thread 的剔除、排序和命令录制工作量增加，无法及时提交命令。
- 成因 2：API 调用的同步开销。OpenGL 的状态机模型导致 API 调用必须串行执行，Vulkan/DX12 的多线程命令录制可以缓解但需要正确的同步。
- 成因 3：资源加载的阻塞。当渲染所需的资源（纹理、Shader）尚未加载完成时，Render Thread 必须等待，GPU 处于空闲状态。

### 避免 GPU Bubble 的核心策略

- 多线程命令录制：使用 Secondary Command Buffer（Vulkan）/ Bundle（D3D12）并行录制不同 Pass 或不同物体的渲染命令，减少 Render Thread 的单线程瓶颈。
- 异步计算（Async Compute）：将计算密集型任务（如 GPU Particle Update、TAA Resolve）放在 Async Compute Queue 上执行，与 Graphics Queue 的渲染并行。
- 帧间并行（Frame Pipelining）：CPU 在处理第 N+1 帧的同时，GPU 在执行第 N 帧。通过双缓冲或三缓冲的 Command Buffer 实现。
- 预提交（Pre-submit）：在 GPU 执行当前帧的同时，CPU 提前录制下一帧的命令并提交到队列中，确保 GPU 始终有命令可执行。


## 🛠 工程实践

### UE 的 Task Graph 和 Render Dependency Graph

- UE 使用 Task Graph 系统管理所有异步任务的调度和依赖关系。Render Thread 通过 Task Graph 派发并行任务给 Worker Thread。
- Render Dependency Graph（RDG）：UE4 引入的渲染 Pass 管理系统，自动管理 Pass 之间的资源依赖（读/写关系），支持 Pass 的并行执行和资源复用。
- RDG 的优势：自动处理 Transient Resource 的生命周期（Pass 之间创建和销毁），减少显存占用；自动推断 Pass 之间的依赖关系，支持并行执行。
- Worker Thread 通过 Task Graph 执行：Frustum Culling、Shadow Cascade 计算、Light Culling、Visibility Buffer 生成等并行任务。

### Vulkan 的 Secondary Command Buffer 并行录制

- Vulkan 支持多线程录制 Secondary Command Buffer，每个 Worker Thread 录制一个或多个 Pass 的命令，最后由 Render Thread 将所有 Secondary Command Buffer 提交到 Primary Command Buffer 中执行。
- 使用方式：vkBeginCommandBuffer（Secondary Level）→ 录制渲染命令 → vkEndCommandBuffer → vkCmdExecuteCommands（Primary Level 中执行）。
- 注意事项：Secondary Command Buffer 不能包含改变全局管线状态（如 Framebuffer 绑定、Render Pass 转换）的命令，这些必须在 Primary Command Buffer 中执行。
- 资源同步：Secondary Command Buffer 之间的资源共享需要使用 Event 或 Barrier 确保正确的执行顺序。

### Fence 和 Semaphore 的同步策略

- Fence：CPU-GPU 同步，CPU 可以等待 Fence（vkWaitForFences）或查询 Fence 状态（vkGetFenceStatus）。用于确保 GPU 完成某帧的渲染后，CPU 才能复用该帧的资源。
- Semaphore：GPU-GPU 同步，用于不同 Queue 之间的执行顺序保证。例如：Graphics Queue 完成渲染后，通过 Semaphore 通知 Compute Queue 可以开始后处理。
- Timeline Semaphore（Vulkan 1.2+）：支持 64-bit 的信号值，可以更灵活地表达复杂的同步关系，减少 Fence 的数量。
- 双缓冲/三缓冲策略：使用 N 个 Command Buffer 轮转（N = 2 或 3），CPU 在录制第 (i+1) 帧 Command Buffer 的同时，GPU 在执行第 i 帧。Fence 确保 CPU 不会覆盖 GPU 正在使用的 Command Buffer。


## ⚠️ 踩坑经验

### 多线程命令录制的资源竞争

- 多个 Worker Thread 并行录制 Command Buffer 时，如果访问共享资源（如全局的 Descriptor Set、Uniform Buffer），可能出现 Race Condition。
- 解决方案：每个 Worker Thread 使用独立的 Descriptor Pool 和 Command Pool，避免共享资源的竞争。
- 只读资源（如纹理、静态 Uniform Buffer）可以安全地被多个 Command Buffer 同时引用，无需额外同步。
- 可变资源（如 Dynamic Uniform Buffer、Storage Buffer）需要使用 Barrier 或 Fence 确保写入和读取的顺序。

### Render Thread 的帧延迟

- 多线程渲染架构引入了帧延迟：Game Thread 在第 N 帧产生的渲染数据，可能在第 N+1 或 N+2 帧才被 GPU 执行。
- 帧延迟导致输入响应延迟（Input Latency）：玩家的输入在 Game Thread 中处理，但渲染结果需要等待额外的帧才能显示。
- 优化方案：减少帧间并行的缓冲数量（从三缓冲改为双缓冲），但可能增加 GPU Bubble 的风险；使用预测渲染（Predictive Rendering）减少感知延迟。

### Worker Thread 负载不均衡

- 如果场景中的物体分布不均匀（如某些区域物体密集、某些区域稀疏），Worker Thread 的剔除和命令录制工作量不均衡，导致部分 Thread 空闲等待。
- 解决方案：使用 Work Stealing 策略——空闲的 Worker Thread 从其他 Thread 的任务队列中\
- 任务执行。
- 使用细粒度的任务划分：将剔除任务按空间划分（如每个 Worker Thread 负责一个屏幕区域），而不是按物体划分（可能导致负载不均衡）。
- 动态调整 Worker Thread 的数量：根据当前帧的工作量动态创建/销毁 Worker Thread，避免不必要的线程创建开销。


## 🔮 延伸思考

### 帧图（Frame Graph）在多线程渲染中的应用

- Frame Graph（Frostbite 引擎提出）是一种声明式的渲染 Pass 管理系统：开发者声明每个 Pass 的输入/输出资源，Frame Graph 自动推断 Pass 之间的依赖关系，调度 Pass 的执行顺序和并行策略。
- Frame Graph 的优势：自动管理 Transient Resource 的生命周期（Pass 之间创建和销毁），自动推断 Pass 之间的依赖关系支持并行执行，支持资源别名（Resource Aliasing）减少显存占用。
- 在多线程环境中，Frame Graph 可以自动识别无依赖关系的 Pass，将它们调度到不同的 Worker Thread 并行执行，最大化 CPU 利用率。
- UE5 的 RDG（Render Dependency Graph）是 Frame Graph 的一种实现，已集成到 UE 的渲染管线中。

### 下一代渲染架构的线程模型

- GPU Driven Rendering 的普及使得 CPU 端的工作量进一步减少，Render Thread 的角色从\
- 转变为\
- 。
- Mesh Shader 和 Task Shader 使得 GPU 可以自主决定几何体的处理方式，CPU 只需要提交高层级的 Dispatch Command。
- WGPU/WebGPU 的异步编程模型：使用 Promise 和 async/await 管理 GPU 操作的异步性，简化多线程渲染的开发。
- 未来的渲染架构可能完全消除 Render Thread，由 GPU 自主管理渲染流程（如 GPU Driven Pipeline + Mesh Shader + Ray Tracing），CPU 只负责高层级的场景管理和资源调度。

---

[← 返回 渲染架构与管线设计 题目列表](./index.md) | [返回总纲](../00-总纲.md)
