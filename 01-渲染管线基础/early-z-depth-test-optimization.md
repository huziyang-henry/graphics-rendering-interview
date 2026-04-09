---
id: Q01.06
title: "什么是Early-Z？它在什么条件下会失效？如何避免Early-Z失效？"
chapter: 1
chapter_name: "渲染管线基础"
difficulty: intermediate
knowledge_points:
  - id: "01.04"
    name: "光栅化原理"
tags: ["early-z", "depth-test", "hi-z", "z-cull"]
---

# Q01.06 什么是Early-Z？它在什么条件下会失效？如何避免Early-Z失效？

**难度：** 🟡 中级
**所属章节：** [渲染管线基础](./index.md)

---

## 🎯 结论

- Early-Z是在片元着色器执行之前进行的深度测试优化，可以提前丢弃被遮挡的片元，避免不必要的FS执行。
- Early-Z在以下条件下失效：FS中修改深度值（gl_FragDepth/SV_Depth）、使用discard/clip()、Alpha Test（旧硬件）、启用Alpha-to-Coverage。
- 避免Early-Z失效的核心原则：保持深度测试的可预测性，让硬件能在FS之前确定深度值。

## 📐 原理解析

### Early-Z的工作机制

- GPU在光栅化生成片元后、执行FS之前，先进行深度测试（Z Test）
- 如果片元的深度值大于Depth Buffer中的值（假设DepthFunc=LESS），则直接丢弃该片元，不执行FS
- Hi-Z（Hierarchical Z）是Early-Z的增强版本：维护一个深度值的Mipmap金字塔，用低分辨率的深度值快速判断整个Tile是否被遮挡
- Z-Cull进一步优化：按Tile粒度判断是否整个Tile都被遮挡，一次性跳过整个Tile的片元处理

### Early-Z失效的条件

- FS中写入gl_FragDepth/SV_Depth：硬件无法在FS执行前知道最终的深度值
- FS中使用discard或clip()：可能丢弃部分片元但不更新深度，破坏了深度缓冲的一致性
- Alpha Test（在旧硬件上）：等价于discard，导致Early-Z失效
- Alpha-to-Coverage：与MSAA交互，可能导致部分样本被丢弃
- 注意：Alpha Blend（混合）不会导致Early-Z失效，因为深度值仍然可以预先确定

### Early-Z Stencil优化

- Early Stencil Test同样可以在FS之前执行，常用于模板阴影（Stencil Shadow Volume）和延迟渲染的Light Volume优化
- Early-Z和Early Stencil可以同时生效，双重剔除效果叠加


## 🛠 工程实践

- 深度Pre-pass（Z Pre-pass）技术：先用一个简单的Shader（只写深度，不执行复杂光照计算）渲染所有不透明物体，填充Depth Buffer。然后在正式渲染Pass中，Early-Z可以高效剔除被遮挡片元。适用于场景复杂度高、Overdraw严重的场景。
- 利用Early-Z优化粒子系统：先渲染不透明物体建立深度缓冲，再渲染半透明粒子时，被遮挡的粒子片元会被Early-Z剔除。
- Depth Bounds Test（深度范围测试）：只接受深度值在[min, max]范围内的片元，可用于限定特定深度区间的渲染效果。
- 在Shader中避免不必要的discard：如果Alpha Test的阈值很低（几乎不丢弃片元），可以考虑改用Alpha Blend或直接关闭Alpha Test以保持Early-Z生效。

## ⚠️ 踩坑经验

- Alpha Test导致Early-Z失效的排查方法：使用RenderDoc查看FS的Invocations和Early-Z Tests的统计，如果两者接近说明Early-Z在生效，如果Invocations远大于Early-Z Tests则说明Early-Z失效。
- Depth Pre-pass的适用性判断：Pre-pass本身也有开销（额外的Draw Call和VS执行），如果场景的Overdraw很低（如2x以下），Pre-pass可能反而降低性能。通常在Overdraw > 3x时才有明显收益。
- 移动端GPU的Early-Z行为差异：部分移动GPU（如早期Mali）的Early-Z实现不如桌面GPU完善，需要通过GPU Profiler验证。
- gl_FragDepth的layout限定符：在GLSL中，使用layout(depth_unchanged)可以告诉编译器深度值不会改变，从而保持Early-Z生效。

### Early-Z、Hi-Z 与 Z-Prepass 的对比与配合

这三种技术都是为了提前剔除不可见片段，减少片段着色器执行次数，但实现层级和适用场景不同：

| 技术 | 实现层级 | 核心原理 | 适用场景 | 性能开销 |
|------|---------|---------|---------|---------|
| Early-Z | 硬件固定功能 | 光栅化阶段先执行深度测试，再执行片段着色器 | 所有不透明物体 | 几乎为 0 |
| Hi-Z | 硬件固定功能 | 生成深度缓冲的 Mipmap，快速剔除整个图元或像素块 | 大尺寸遮挡物后的物体 | 极低 |
| Z-Prepass | 软件渲染 Pass | 先渲染一遍深度（不输出颜色），再渲染颜色 Pass | 复杂片段着色器、高 Overdraw 场景 | 额外一次顶点处理 |

**工程配合策略**：
1. 所有不透明物体默认开启 Early-Z（硬件免费）
2. 开启 Hi-Z 配合硬件遮挡剔除，快速剔除被大物体遮挡的整个物体
3. 当 Overdraw 很高（复杂 PBR 材质、大量半透明物体）时使用 Z-Prepass——只写入深度不执行片段着色器，颜色 Pass 时 Early-Z 剔除被遮挡片段
4. Alpha Test 和 Alpha Blend 物体不能使用 Early-Z（需执行片段着色器才能决定是否丢弃）

**踩坑经验**：Z-Prepass 增加 Draw Call 数量，需配合合批使用；移动端 TBR 架构下 Z-Prepass 收益比 PC 更大，因为减少片段着色器执行能大幅降低带宽。

## 🔮 延伸思考

- Depth Pre-pass在什么情况下反而降低性能？当场景的Overdraw很低、VS计算量很大（如骨骼动画角色）、或者GPU是Compute Bound而非Bandwidth Bound时，Pre-pass的额外开销可能超过其收益。
- Forward+（Tiled Forward）渲染中，每个Tile的光照列表是在FS中动态查询的，这意味着FS执行前无法完全确定哪些光照会影响片元。但深度值本身仍然是可预测的，所以Early-Z仍然有效。
- 延迟渲染（Deferred Rendering）天然避免了Overdraw问题：G-Buffer Pass中每个像素只写一次，Light Pass中通过Depth Buffer进行范围查询。这是延迟渲染在复杂场景中性能优于前向渲染的重要原因之一。

---

[← 返回 渲染管线基础 题目列表](./index.md) | [返回总纲](../00-总纲.md)
