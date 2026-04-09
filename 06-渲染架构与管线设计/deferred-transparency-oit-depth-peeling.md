---
id: Q06.03
title: "延迟渲染为什么不能很好地处理半透明物体？常见的解决方案有哪些？"
chapter: 6
chapter_name: "渲染架构与管线设计"
difficulty: intermediate
knowledge_points:
  - id: "06.03"
    name: "半透明物体渲染方案"
tags: ["transparency", "oit", "depth-peeling", "wboit"]
---

# Q06.03 延迟渲染为什么不能很好地处理半透明物体？常见的解决方案有哪些？

**难度：** 🟡 中级
**所属章节：** [渲染架构与管线设计](./index.md)

---

## 🎯 结论

- 半透明物体需要按从后到前的顺序混合（Back-to-Front Blending），每个像素可能需要累积多个半透明层的颜色贡献。
- 延迟渲染的 Lighting Pass 假设每个像素只有一个可见片元（深度测试获胜者），G-Buffer 只存储了最近表面的信息，无法表示多个半透明层的叠加。
- 因此延迟渲染管线中，半透明物体必须回退到 Forward 渲染，或使用专门的 Order-Independent Transparency（OIT）算法。

## 📐 原理解析

### 延迟渲染与半透明的不兼容性

- 延迟渲染的 G-Buffer Pass 使用深度测试（Depth Test, LEQUAL），每个像素只保留深度最小的片元的几何和材质信息。
- 半透明物体的渲染需要 Alpha Blending：C_out = C_src × α_src + C_dst × (1 - α_src)，这要求按从后到前的顺序逐层累积。
- 如果将半透明物体写入 G-Buffer，只会保留最近的一层，后续层的颜色信息丢失，导致半透明效果完全错误。
- 此外，半透明物体可能需要读取其背后的不透明表面的信息（如折射、色散），而 G-Buffer 只存储了不透明表面的信息。

### Alpha Blending 的顺序依赖性

- 标准 Alpha Blending 的结果依赖于片元的绘制顺序：先绘制远处的半透明物体，再绘制近处的，才能得到正确的颜色累积。
- 在延迟渲染中，G-Buffer Pass 的绘制顺序由引擎的排序策略决定，但半透明物体之间的正确排序需要精确的画家算法（Painter's Algorithm），这在复杂场景中难以保证。
- 即使使用 Depth Peeling 或 OIT，也需要额外的 Pass 或内存开销来存储和排序所有半透明层。


## 🛠 工程实践

### 半透明物体回退到 Forward 渲染

- 最常见的工业方案：在 Deferred 渲染管线中，不透明物体使用 Deferred，半透明物体切换到 Forward 渲染。
- 实现方式：在 G-Buffer Pass 后，禁用 Depth Write，启用 Alpha Blending，按从后到前的顺序渲染半透明物体。
- 半透明物体的光照计算在 Forward Shader 中完成，可以使用 G-Buffer 中的深度和法线信息作为辅助输入（如屏幕空间反射、环境光遮蔽）。
- 优点：实现简单，与现有管线兼容性好。缺点：半透明物体的光源数量受限（需要遍历所有光源），且无法享受 Deferred 的屏幕空间后处理。

### Depth Peeling（逐层剥离）

- 多次渲染 Pass，每次剥离当前最近的半透明层：第一层渲染所有半透明物体，提取深度最小的片元；第二层渲染深度大于第一层的片元，以此类推。
- 需要的 Pass 数量等于半透明层的最大深度复杂度（Depth Complexity），通常限制在 4-8 层以控制性能。
- 适用于半透明层数较少且已知的场景（如水面、玻璃窗）。

### Order-Independent Transparency（OIT）

- Weighted Blended OIT（WBOIT）：为每个片元计算权重（基于透明度和深度），使用加法混合累积颜色和权重，最后除以总权重得到最终颜色。优点：单 Pass 实现，性能好。缺点：结果近似，不精确。
- Per-Pixel Linked List（PPLL）：使用 Atomic Counter 和 Shader Storage Buffer 在每个像素上构建一个链表，存储所有半透明片元，最后在 Resolve Pass 中排序并混合。优点：结果精确。缺点：显存开销大，链表构建的性能开销高。
- Multi-Layer Alpha Blending：利用硬件支持的 Dual-Source Blending 或 Fragment Shader Interlock，在有限层数内实现精确的 OIT。


## ⚠️ 踩坑经验

### OIT 的内存开销和排序精度

- PPLL 的显存开销与屏幕分辨率和最大半透明层数成正比。在 1080p 下，假设每像素最大 16 层，每层 32 字节（颜色 + 深度 + 下一节点指针），总开销约 1920×1080×16×32 ≈ 1GB。
- WBOIT 的近似结果在半透明物体互相重叠时可能出现颜色偏差，特别是在高透明度物体叠加低透明度物体时。
- OIT 的排序精度受限于浮点数的精度，深度值接近的片元可能出现排序错误。

### Depth Peeling 的 Pass 数量与性能的关系

- Depth Peeling 的性能与半透明层数线性相关。在复杂场景中（如粒子系统、体积雾），层数可能很大，导致性能严重下降。
- 固定最大层数（如 4 层）可以控制性能，但可能导致远处的半透明层被丢弃，产生视觉瑕疵。
- Early-Z 优化可以减少不必要的片元处理，但在半透明渲染中 Depth Write 被禁用，Early-Z 无法生效。

### 半透明物体的阴影处理

- 半透明物体投射的阴影应该是柔和的、带有颜色调制的（如彩色玻璃），但标准 Shadow Map 只存储深度信息。
- 解决方案：使用 Translucent Shadow Map（存储颜色和透明度）或基于 SDF（Signed Distance Field）的软阴影。
- 半透明物体接收阴影也需要特殊处理：G-Buffer 中不包含半透明物体的深度信息，Shadow Pass 需要额外考虑半透明物体的遮挡。


## 🔮 延伸思考

### 光线追踪对半透明渲染的简化

光线追踪天然支持半透明渲染：光线穿过半透明物体时，根据材质的传输属性（Transmission、Absorption）递归地追踪折射和反射光线。每次光线与半透明表面的交互都遵循物理正确的折射定律（Snell's Law）和菲涅尔方程。这消除了传统光栅化中排序和混合的复杂性，但计算开销远大于光栅化方案。在混合渲染管线（Ray Tracing + Rasterization）中，可以用光线追踪处理关键半透明物体（如玻璃、水面），用光栅化处理其余场景。

### 实用的半透明渲染策略

- 分层策略：将半透明物体分为\
- （如玻璃、水面，使用精确 OIT 或光线追踪）和\
- （如粒子、烟雾，使用 WBOIT 或简单的加法混合），分别处理。
- Dithered Transparency：将半透明物体以抖动（Dither）方式渲染为不透明，避免排序问题。适用于植被、毛发等大量细小半透明片元的场景。
- Pre-Integrated Transparency：对半透明材质的透射光进行预积分，在 Fragment Shader 中查表获取透射颜色，减少逐片元的计算开销。

---

[← 返回 渲染架构与管线设计 题目列表](./index.md) | [返回总纲](../00-总纲.md)
