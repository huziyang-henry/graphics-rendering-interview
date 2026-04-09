---
id: Q01.09
title: "现代GPU的渲染管线中，Tile-Based Rendering（TBR/TBDR）与传统Immediate Mode Rendering在架构上有什么本质区别？"
chapter: 1
chapter_name: "渲染管线基础"
difficulty: expert
knowledge_points:
  - id: "01.08"
    name: "TBR/TBDR与IMR架构"
tags: ["tbr", "tbdr", "imr", "mobile-gpu"]
---

# Q01.09 现代GPU的渲染管线中，Tile-Based Rendering（TBR/TBDR）与传统Immediate Mode Rendering在架构上有什么本质区别？

**难度：** 🟣 专家级
**所属章节：** [渲染管线基础](./index.md)

---

## 🎯 结论

- TBR/TBDR将屏幕划分为固定大小的Tile（如16x16或32x32像素），每个Tile在片上高速缓存（On-Chip Tile Memory）中完成所有渲染操作，最终一次性写入主存。
- 传统IMR（Immediate Mode Rendering）逐Draw Call直接渲染到主存中的Frame Buffer，每次读写都需要访问高延迟的DRAM。
- TBDR相比IMR的最大优势是大幅降低带宽消耗（通常减少3-5倍），这是移动端GPU普遍采用TBDR架构的根本原因。

## 📐 原理解析

### IMR（Immediate Mode Rendering）架构

- 工作流程：接收Draw Call → 逐三角形光栅化 → 逐片元执行FS → 直接读写Frame Buffer（DRAM）
- 每个Draw Call的渲染结果立即写入主存，后续Draw Call读取时需要再次访问DRAM
- 典型代表：桌面端NVIDIA和AMD的独立GPU
- 优势：实现简单，延迟低（单Draw Call的延迟可预测）
- 劣势：带宽消耗大，Overdraw场景下同一像素可能被读写多次

### TBR（Tile-Based Rendering）架构

- 工作流程：接收所有Draw Call → 几何处理阶段（Binning：将三角形分配到Tile）→ 逐Tile渲染（在片上缓存中完成所有Draw Call的渲染）→ 写回DRAM
- Binning阶段：将每个三角形标记到它覆盖的Tile列表中
- 渲染阶段：对每个Tile，依次执行所有覆盖该Tile的Draw Call的FS，结果暂存片上缓存
- 最终Tile内完成深度测试和混合后，一次性写回DRAM
- 典型代表：ARM Mali、Qualcomm Adreno、Imagination PowerVR

### TBDR（Tile-Based Deferred Rendering）架构

- TBDR是TBR的增强版本：在Tile渲染前先执行Hidden Surface Removal（HSR），在片上缓存中剔除被遮挡的片元
- HSR可以在FS执行前确定每个像素的最终可见片元，完全消除被遮挡片元的FS执行
- 这是TBDR相比TBR的额外优势：不仅减少带宽，还减少计算量
- 典型代表：Apple A系列和M系列芯片中的GPU


## 🛠 工程实践

- 移动端Load/Store Action优化：TBR架构中，每个Render Target在开始渲染时需要从DRAM加载到片上缓存（Load），渲染完成后需要写回（Store）。使用glInvalidateFramebuffer/discard可以跳过不必要的Load/Store操作。
- 避免破坏TBDR效率的渲染模式：不要在渲染中间读取Frame Buffer内容（如glReadPixels、glCopyTexImage2D），这会强制Tile数据写回DRAM再重新加载。
- 减少Render Target的切换：每次切换RT都需要执行当前RT的Store和新RT的Load。合并使用相同RT的渲染Pass可以减少Load/Store次数。
- Apple的Metal API提供了explicit tile shading API，允许开发者直接控制Tile的渲染流程，进一步优化带宽。

## ⚠️ 踩坑经验

- 在TBDR架构上使用IMR思维的开发陷阱：在桌面端优化良好的渲染代码在移动端可能表现很差。例如，频繁的Framebuffer Readback在TBDR上会导致严重的性能惩罚。
- MSAA在TBDR上的特殊行为：MSAA的覆盖率信息存储在片上缓存中，不需要额外的带宽。这意味着移动端开启MSAA的带宽开销比桌面端小得多，但计算开销（更多FS执行）仍然存在。
- Debug工具的差异：RenderDoc在移动端上的Capture可能无法完全反映TBDR的Tile执行顺序，需要结合GPU-specific Profiler（如Mali Graphics Debugger、Snapdragon Profiler）进行分析。
- 部分桌面GPU也采用了类似TBR的优化：NVIDIA的Tile-Based Cache和AMD的Delta Color Compression都是在IMR基础上增加Tile级别的缓存优化，但本质上仍然是IMR架构。

## 🔮 延伸思考

- Apple Silicon的GPU架构创新：Apple的M系列芯片GPU采用了更先进的TBDR架构，支持更大的Tile和更高效的HSR。结合统一内存架构（Unified Memory Architecture），CPU和GPU共享同一内存池，消除了数据拷贝开销。
- PC端GPU是否会走向TBDR？短期内不太可能完全转变，因为IMR在低延迟和灵活性方面仍有优势。但混合架构（如NVIDIA的GDDR片上缓存）正在借鉴TBR的思想。Intel Arc GPU采用了类似TBR的架构。
- 对开发者而言，理解目标平台的GPU架构对于性能优化至关重要。跨平台引擎需要针对IMR和TBDR分别优化渲染策略，或者采用对两种架构都友好的通用策略。

---

[← 返回 渲染管线基础 题目列表](./index.md) | [返回总纲](../00-总纲.md)
