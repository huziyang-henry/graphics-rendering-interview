---
id: Q08.06
title: "什么是可变速率着色（Variable Rate Shading, VRS）？它如何与眼球追踪技术结合以提升VR渲染性能？"
chapter: 8
chapter_name: "高级渲染技术与前沿方向"
difficulty: expert
knowledge_points:
  - id: "08.05"
    name: "可变速率着色（VRS）与VR优化"
tags: ["vrs", "foveated-rendering", "vr", "eye-tracking", "shading-rate"]
---

# Q08.06 什么是可变速率着色（Variable Rate Shading, VRS）？它如何与眼球追踪技术结合以提升VR渲染性能？

**难度：** 🟣 专家级
**所属章节：** [高级渲染技术与前沿方向](./index.md)

---

## 🎯 结论

- 可变速率着色（Variable Rate Shading, VRS）是一种GPU级别的渲染优化技术，允许在不同屏幕区域使用不同的着色频率（Shading Rate）。在人眼注视区域保持全分辨率着色，在周边视觉区域降低着色频率，从而在几乎不影响视觉感知的情况下大幅减少着色计算量。当VRS与眼球追踪结合时（Foveated Rendering），可以实现精确的注视点渲染，在VR场景中可获得30-50%的性能提升，是解决VR高分辨率渲染性能瓶颈的关键技术之一。

## 📐 原理解析

### VRS的基本原理

- 传统渲染中，每个像素独立计算着色结果（1x1 Shading Rate），VRS允许将多个像素共享同一次着色计算
- 支持的Shading Rate：1x1（每像素独立着色）、1x2（两个水平像素共享）、2x1（两个垂直像素共享）、2x2（2x2像素块共享）、2x4、4x2、4x4等
- 2x2 Shading Rate意味着着色计算量减少为1/4，但图像分辨率不变（只是着色频率降低），因此不会影响UI和几何体的清晰度
- VRS通过Shading Rate Image（SRI）控制每个屏幕区域的着色频率，SRI是一个低分辨率的纹理，每个Texel对应屏幕上的一个Tile（通常8x8或16x16像素）
- VRS作用于Fragment Shader阶段，不影响几何处理（顶点着色、光栅化），因此几何体的边缘精度不受影响

### VRS的实现方式（API层面）

- DX12的Variable Rate Shading Tier：Tier 1（仅基于屏幕空间的粗粒度VRS）、Tier 2（基于Shading Rate Image的细粒度VRS）
- Vulkan的VK_KHR_fragment_shading_rate扩展：提供类似的VRS功能，支持Per-Draw、Per-Primitive和Per-Attachment三种模式
- VRS可以由以下方式驱动：应用程序手动设置（Procedural VRS）、基于眼球追踪（Eye-Tracked VRS）、基于内容分析（Content-Adaptive VRS）
- Shading Rate的组合：当多个VRS来源同时存在时，GPU取最小值（最低着色频率），确保不会超过任何来源设定的限制

### 眼球追踪与注视点渲染（Foveated Rendering）

- 人眼的视觉特性：中央凹（Fovea）区域具有最高的空间分辨率，周边视觉的分辨率急剧下降
- 注视点渲染利用这一特性：在用户注视的区域使用全分辨率渲染，在周边区域使用低分辨率渲染
- 眼球追踪技术（Eye Tracking）提供实时的注视点位置（Gaze Point），精度通常为0.5-1度视角
- 将注视点位置映射到Shading Rate Image：注视中心为1x1，向外逐渐降低到2x2、4x4，形成径向渐变的着色频率分布
- 典型的注视点渲染配置：中心5-10度视角为1x1，10-30度为1x2或2x2，30度以外为4x4，整体着色计算量可减少40-60%


## 🛠 工程实践

### VRS与眼球追踪在VR头显中的应用

- Meta Quest Pro / Quest 3：支持基于眼球追踪的Foveated Rendering，配合高通Adreno GPU的VRS扩展
- PSVR2：内置眼球追踪，支持硬件级别的注视点渲染，开发者通过PSVR2 API获取注视点数据并设置VRS
- Varjo XR-3/XR-4：专业级VR头显，支持高精度眼球追踪（亚像素级）和注视点渲染，注视区域可精确到1度以内
- 实现步骤：初始化眼球追踪 → 获取注视点坐标 → 生成Shading Rate Image → 设置VRS → 渲染场景

### 基于内容分析的自动VRS（无眼球追踪）

- 在没有眼球追踪硬件的平台上，可以通过内容分析自动生成Shading Rate Image
- 基于运动向量（Motion Vector）：快速运动的区域使用低着色频率（人眼对运动物体的细节不敏感），静态区域使用高频率
- 基于深度梯度（Depth Gradient）：深度变化小的区域（如墙壁、地面）使用低频率，深度变化大的区域（如物体边缘）使用高频率
- 基于亮度/对比度：低对比度区域使用低频率，高对比度区域使用高频率
- UE5的VRS支持：通过r.VRS.Enable控制台变量启用，支持基于运动向量和深度梯度的自动VRS

### VRS的性能收益与调优

- VRS的性能收益取决于Fragment Shader的开销占比：如果场景是Compute-bound或Vertex-bound，VRS的收益有限
- 对于Fragment-bound的场景（复杂材质、多重采样、延迟渲染的Lighting Pass），VRS可以带来显著的性能提升
- 调优建议：分析GPU Profiler确定瓶颈阶段，对Fragment-bound的Pass启用VRS；对非Fragment-bound的Pass（如Depth Pre-pass）禁用VRS
- VRS与超分辨率（DLSS/FSR）可以叠加使用：先VRS降低着色频率，再超分辨率提升输出分辨率，获得双重性能收益


## ⚠️ 踩坑经验

### VRS导致的边缘模糊和细节丢失

- 在Shading Rate大于1x1的区域，相邻像素共享着色结果，导致高频细节（细线条、文字、纹理细节）丢失或模糊
- 特别是UI元素（HUD、文字、小图标）对VRS非常敏感，低着色频率会导致文字不可读
- 解决方案：对UI元素禁用VRS（使用1x1 Shading Rate渲染UI到单独的Pass）；或在Shading Rate Image中将UI区域标记为1x1
- 精细纹理（如布料纹理、毛发细节）在高Shading Rate下可能出现莫尔纹（Moire Pattern），需要降低纹理的Mip Level

### VRS与后处理效果的交互问题

- 某些后处理效果对VRS敏感：SSAO、Motion Blur、Bloom等在低着色频率区域可能出现伪影
- 延迟渲染的G-Buffer Pass不应使用VRS（法线、深度需要全精度），仅在Lighting Pass和后续Pass使用VRS
- 抗锯齿（MSAA）与VRS的交互：MSAA的覆盖率计算在低着色频率区域可能不准确，建议使用TAA而非MSAA
- VRS可能影响Alpha Test的精度：在Shading Rate为4x4时，Alpha Test的边缘会出现明显的锯齿

### 不同GPU对VRS的支持粒度差异

- NVIDIA Turing及以后架构：支持Tier 2 VRS（Shading Rate Image），Tile大小为8x8像素
- AMD RDNA2及以后架构：支持VRS，Tile大小为8x8像素，但某些型号的精度可能低于NVIDIA
- Intel Arc架构：支持VRS，但驱动支持和性能表现可能不如NVIDIA/AMD成熟
- 移动端GPU（Adreno、Mali）：VRS支持因型号而异，需要查询设备能力并设置合理的回退方案


## 🔮 延伸思考

### VRS在非VR场景中的应用潜力

- 虽然VRS最初为VR设计，但在非VR场景中也有广泛应用价值
- 开放世界游戏：远处景物使用低着色频率，近处使用高频率，配合LOD系统进一步优化性能
- 竞速游戏：基于速度和注视方向的VRS，高速运动时降低周边着色频率
- 策略游戏：在小地图/UI密集区域保持高频率，在3D场景区域使用低频率
- VRS可以作为通用的性能优化手段，在几乎所有3D应用中都能获得一定收益

### VRS与超分辨率技术的协同

- VRS和超分辨率（DLSS/FSR）都通过降低着色计算量来提升性能，但机制不同：VRS降低着色频率，超分辨率降低渲染分辨率
- 两者可以叠加：先在低分辨率渲染中使用VRS进一步降低着色频率，再通过超分辨率放大到输出分辨率
- 潜在问题：双重降质量可能导致画质明显下降，需要仔细平衡VRS的Shading Rate和超分辨率的缩放因子
- 未来方向：DLSS等AI超分辨率方案可能内置VRS优化，AI网络自动学习哪些区域可以降低着色频率而不影响输出质量

---

[← 返回 高级渲染技术与前沿方向 题目列表](./index.md) | [返回总纲](../00-总纲.md)
