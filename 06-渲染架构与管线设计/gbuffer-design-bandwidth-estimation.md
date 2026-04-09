---
id: Q06.02
title: "延迟渲染中G-Buffer通常包含哪些内容？G-Buffer的带宽开销如何估算？"
chapter: 6
chapter_name: "渲染架构与管线设计"
difficulty: intermediate
knowledge_points:
  - id: "06.02"
    name: "G-Buffer设计与优化"
tags: ["gbuffer", "bandwidth", "mrt", "render-target"]
---

# Q06.02 延迟渲染中G-Buffer通常包含哪些内容？G-Buffer的带宽开销如何估算？

**难度：** 🟡 中级
**所属章节：** [渲染架构与管线设计](./index.md)

---

## 🎯 结论

- G-Buffer 是延迟渲染的核心数据结构，存储了后续 Lighting Pass 计算光照所需的全部几何和材质信息。
- 典型内容包含：World Position（或 Depth 用于重建）、World Normal、Base Color（Albedo）、Metallic、Roughness、AO、Emissive、Motion Vector 等。
- 带宽开销估算公式：总带宽 = Render Target 数量 $\times$ 屏幕分辨率 $\times$ 每像素字节数 $\times$ 帧率。在 1080p@60fps 下，典型 G-Buffer 带宽可达 10-20 GB/s。

## 📐 原理解析

### G-Buffer 各属性的作用

- World Position / Depth：用于计算光源到片元的距离和方向。Depth 可以通过逆投影矩阵重建 World Position，节省一个 RT。
- World Normal：用于计算光照的 $\mathbf{N} \cdot \mathbf{L}$ 和 $\mathbf{N} \cdot \mathbf{V}$ 项，是 PBR 光照计算的核心输入。
- Base Color（Albedo）：物体的漫反射颜色，不含光照信息。
- Metallic：区分金属和非金属材质，影响 F0（菲涅尔反射率）的计算。
- Roughness：控制镜面反射的粗糙程度，影响高光分布（GGX/Beckmann 分布）。
- AO（Ambient Occlusion）：环境光遮蔽因子，通常由 SSAO 或预烘焙提供。
- Emissive：自发光颜色，直接叠加到最终输出，不参与光照计算。
- Motion Vector：用于 TAA 的运动模糊和历史帧采样。

### G-Buffer 的存储策略

- 独立 RT 方案：每个属性占用一个独立的 Render Target，访问简单但 RT 数量多（可能超过硬件限制）。
- Packing 方案：将多个属性打包到单个 RT 的不同通道中。例如 Normal.xy + Roughness 打包到 RGBA16F 的 RGB 通道，A 通道存储 Metallic。
- 带宽估算：假设 4 个 RGBA16F RT（每像素 $4 \times 8 = 32$ 字节），1080p 分辨率（ $1920 \times 1080 \approx 207$ 万像素），每帧写入带宽 = $32 \times 2070000 \approx 66\text{MB}$ 。加上 Lighting Pass 的读取带宽（同样约 66MB），仅 G-Buffer 的读写带宽就达到约 132MB/帧，60fps 下约 7.9 GB/s。


## 🛠 工程实践

### 常见的 G-Buffer 布局方案

- 4-RT 方案（UE4 默认）：RT0 = Base Color（RGBA8），RT1 = World Normal（RG16F）+ Roughness（B16F），RT2 = World Position（RGB16F）或 Depth，RT3 = Metallic（R8）+ AO（G8）+ CustomData（BA8）。
- 3-RT 方案：用 Depth 重建 Position（节省一个 RT），将 Normal 编码到两个通道中（如八面体编码到 RG），其余属性 Packing 到剩余空间。
- Visibility Buffer 方案（UE5）：只存储 Primitive ID（uint）和 Instance ID（uint），在 Lighting Pass 中根据 ID 查询材质属性。极大减少 G-Buffer 带宽，但增加了材质查询的复杂度。

### Position 用 Depth 重建

- 从 Depth Buffer 重建 World Position： $\mathbf{P}_{\text{world}} = \text{InvViewProj} \times (\text{clip}_x, \, \text{clip}_y, \, \text{depth}, \, 1)$ ，其中 clip_x 和 clip_y 可由屏幕 UV 计算。
- 节省一个 RGB16F 或 RGB32F 的 Render Target，显著降低带宽和显存占用。
- 重建计算在 Lighting Pass 的 Fragment Shader 中执行，增加了少量 ALU 开销，但远小于带宽节省带来的收益。

### Normal 的编码方式

- 世界空间直接存储：使用 RG16F 存储 Normal.xy，Normal.z 通过 $\sqrt{1 - x^2 - y^2}$ 重建。简单但精度在 z 轴方向上分布不均匀。
- 切线空间存储：存储切线空间下的 Normal，在 Lighting Pass 中需要 Tangent Frame 重建世界空间法线。适合骨骼动画物体。
- 八面体编码（Octahedral Encoding）：将单位球面映射到二维八面体表面，再用 RG8 或 RG16 存储。精度分布更均匀，是当前主流方案。


## ⚠️ 踩坑经验

### G-Buffer 精度不足导致的法线带状伪影

- 使用 RGBA8 存储法线时，8-bit 精度只有 256 个离散值，在光滑表面上会产生明显的带状伪影（Band Artifacts）。
- 解决方案：使用至少 RGB10A2（10-bit 精度，1024 个离散值）或 RGBA16F（16-bit 半精度浮点）存储法线。
- 八面体编码配合 RG16F 可以在两个通道中存储足够精度的法线信息。

### Packing/Unpacking 的计算开销与带宽节省的权衡

- Packing 将多个属性压缩到更少的 RT 中，减少了带宽和显存占用，但增加了 Packing（Geometry Pass）和 Unpacking（Lighting Pass）的 ALU 开销。
- 在带宽受限的场景（高分辨率、移动端），Packing 的收益通常远大于 ALU 开销。
- 在计算受限的场景（低端 GPU、复杂材质），过多的 Packing/Unpacking 可能成为瓶颈。需要通过 Profiling 确定最优方案。


## 🔮 延伸思考

### G-Buffer 压缩技术

- GPU 驱动级别的 Render Target 压缩（Lossless Delta Compression）：现代 GPU（NVIDIA、AMD）自动对 G-Buffer RT 进行无损压缩，减少显存带宽。
- G-Buffer 分辨率缩放：在动态分辨率或 Checkerboard Rendering 下，G-Buffer 可以低于最终输出分辨率，进一步减少带宽。
- 基于 AI 的 G-Buffer 压缩：利用神经网络对 G-Buffer 进行有损压缩和重建，目前处于研究阶段。

### WebGL/WebGPU 环境下的 G-Buffer 限制

- WebGL 2.0 的 MRT（Multiple Render Targets）数量限制通常为 8 个，但实际可用数量取决于浏览器和硬件实现。
- 浮点纹理（FLOAT/HALF_FLOAT）在 WebGL 中需要启用 OES_texture_float / OES_texture_half_float 扩展，且不是所有设备都支持线性过滤。
- WebGPU 提供了更灵活的 Render Bundle 和 Storage Texture 支持，G-Buffer 的实现更加灵活，但带宽优化仍然重要。

---

[← 返回 渲染架构与管线设计 题目列表](./index.md) | [返回总纲](../00-总纲.md)
