---
id: Q07.08
title: "请解释Overdraw的概念。如何量化Overdraw？在高Overdraw场景下，延迟渲染和前向渲染哪个更优？为什么？"
chapter: 7
chapter_name: "性能优化与GPU架构"
difficulty: intermediate
knowledge_points:
  - id: "07.06"
    name: "Overdraw分析与优化"
tags: ["overdraw", "fill-rate", "bandwidth"]
---

# Q07.08 请解释Overdraw的概念。如何量化Overdraw？在高Overdraw场景下，延迟渲染和前向渲染哪个更优？为什么？

**难度：** 🟡 中级
**所属章节：** [性能优化与GPU架构](./index.md)

---

## 🎯 结论

- Overdraw是指同一屏幕像素在单帧内被片元着色器（FS）多次执行的现象——后渲染的片元覆盖先渲染的片元，之前执行的FS计算全部被浪费。Overdraw直接导致FS计算量和带宽消耗成倍增加，是移动端和复杂场景中最常见的性能问题之一。在高Overdraw场景下，延迟渲染通常优于前向渲染，因为延迟渲染的G-Buffer每个像素只写一次，避免了重复计算物体表面属性的开销。

## 📐 原理解析

### Overdraw的量化定义

- Overdraw倍数 = 实际执行的FS调用次数 / 屏幕像素总数。例如1920x1080分辨率下，如果FS总共执行了12,441,600次（1920*1080*6），则Overdraw为6x
- 理想情况下Overdraw为1x（每个像素只被FS执行一次），但实际场景中由于物体重叠和半透明物体，Overdraw通常为2-10x
- 高Overdraw场景：粒子系统（可达10-100x）、茂密植被（3-10x）、复杂室内场景（2-5x）、多层UI叠加

### Overdraw的性能影响

Overdraw浪费的资源包括：FS计算（被覆盖像素的光照计算全部浪费）、带宽（被覆盖像素的RT写入全部浪费）、深度测试（即使Early-Z拒绝了被覆盖像素的写入，深度测试本身仍有开销）。在移动端，带宽浪费的影响尤为严重，因为移动端GPU的带宽远低于桌面端。


## 🛠 工程实践

### 使用RenderDoc量化Overdraw

RenderDoc提供了Overdraw可视化工具，将每个像素的Overdraw倍数映射为颜色（绿色=1x，黄色=2-3x，红色=4x+）。通过Capture一帧后查看Overdraw视图，可以直观地定位Overdraw热点区域。这是量化Overdraw最常用的方法。

### 通过排序减少Overdraw

对于不透明物体，按从前到后（Front-to-Back）的顺序渲染可以最大化Early-Z的效率——先渲染的物体写入深度缓冲，后续被遮挡的物体在Early-Z阶段就被拒绝，FS不会执行。注意：前向渲染中需要手动或通过引擎设置排序；延迟渲染中G-Buffer Pass通常不需要排序（因为不执行光照计算）。

### 使用Pre-Z Pass减少Overdraw

Pre-Z Pass（也称Depth Pre-Pass或Z-Prepass）是在正式渲染前先用一个简化Shader（只写深度，不执行光照和纹理采样）渲染所有不透明物体，建立完整的深度缓冲。然后在正式渲染Pass中，Early-Z可以高效剔除被遮挡的片元。代价是额外的Draw Call开销（每个物体渲染两次），但在高Overdraw场景中收益远大于开销。


## ⚠️ 踩坑经验

### 半透明粒子的Overdraw最难控制

半透明物体必须按从后到前（Back-to-Front）的顺序渲染以保证正确的混合结果，这意味着Early-Z无法生效——所有半透明片元的FS都会执行。粒子系统是Overdraw最严重的场景，一个火焰效果可能包含数百个半透明粒子，中心区域的Overdraw可达50x以上。缓解方法包括：使用Additive Blending（可以不需要排序）、使用预渲染的Sprite Sheet替代实时粒子、限制粒子数量和大小。

### 延迟渲染Light Pass的Overdraw

延迟渲染的Light Pass中，每个光源对覆盖范围内的所有像素执行光照计算，即使这些像素最终的光照贡献很小。在多光源场景中，Light Pass的Overdraw可能非常显著（如10个重叠光源的区域，每个像素的光照计算执行10次）。优化方法包括：Tiled/Clustered Deferred Shading（将屏幕分块，只处理影响该Tile的光源）、Light Culling（剔除贡献低于阈值的光源）。


## 🔮 延伸思考

### 延迟渲染在Overdraw方面的优势与劣势

延迟渲染的优势：G-Buffer Pass中每个像素只写一次（不考虑Overdraw的影响），物体表面属性（Albedo、Normal等）的计算只执行一次。前向渲染中，被覆盖像素的光照计算全部浪费。延迟渲染的劣势：Light Pass的Overdraw——每个光源覆盖的像素都要执行光照计算，且需要读取G-Buffer（带宽开销）。总体而言，在物体数量多、光源数量中等的场景中，延迟渲染的Overdraw效率优于前向渲染。

### TBDR的HSR对Overdraw的消除

TBDR（Tile-Based Deferred Rendering）架构（如Mali、Apple GPU）在每个Tile渲染前执行Hidden Surface Removal（HSR），在片元着色器执行前就确定每个像素的最终可见片元。这意味着被遮挡片元的FS完全不会执行，从根本上消除了不透明物体的Overdraw。这是TBDR架构在高Overdraw场景下相比传统IMR（Immediate Mode Rendering）架构的最大优势。

---

[← 返回 性能优化与GPU架构 题目列表](./index.md) | [返回总纲](../00-总纲.md)
