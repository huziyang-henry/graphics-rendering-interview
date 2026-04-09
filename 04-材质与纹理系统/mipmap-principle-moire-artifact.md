---
id: Q04.01
title: "Mipmap的原理是什么？它如何解决纹理缩小时的摩尔纹（Moire）问题？Mipmap会带来多少额外的显存开销？"
chapter: 4
chapter_name: "材质与纹理系统"
difficulty: beginner
knowledge_points:
  - id: "04.01"
    name: "纹理采样与过滤（Mipmap/AF）"
tags: ["mipmap", "moire", "texture-sampling", "lod"]
---

# Q04.01 Mipmap的原理是什么？它如何解决纹理缩小时的摩尔纹（Moire）问题？Mipmap会带来多少额外的显存开销？

**难度：** 🟢 初级
**所属章节：** [材质与纹理系统](./index.md)

---

## 🎯 结论

- Mipmap（MIP = Multum In Parvo，意为"小空间中的多事物"）是纹理的预计算多级分辨率金字塔结构，用于解决纹理缩小时因采样频率不足导致的摩尔纹（Moire Pattern）和闪烁（Aliasing）问题。
- Mipmap会带来约33%的额外显存开销。对于一张NxN的纹理，完整的mipmap chain总像素量为 N*N * (1 + 1/4 + 1/16 + ...) = N*N * 4/3，即额外增加约1/3的存储空间。

## 📐 原理解析

### Mipmap的生成原理

- Mipmap Chain由一系列逐级降采样的纹理层级组成，Level 0为原始纹理，Level 1分辨率为原始的1/2（宽高各减半），Level 2为1/4，以此类推，直到1x1像素。
- 每一级通过对上一级的2x2像素区域取平均值生成（Box Filter），确保低层级纹理是高层级的正确低通滤波结果。
- 对于非2的幂纹理，GPU在生成mipmap时会自动处理边界情况，但非2的幂纹理在某些旧硬件上可能不支持完整mipmap chain。

### LOD（Level of Detail）自动选择机制

- GPU在片元着色器中通过纹理坐标的屏幕空间偏导数（ddx(uv)和ddy(uv)）计算当前像素对应的纹理变化率。
- 变化率越大（纹理被缩小得越多），GPU自动选择越高层级的mipmap；变化率越小（纹理被放大），则使用Level 0。
- LOD计算公式：LOD = log2(max(|ddx(uv)|, |ddy(uv)|) * textureSize)。GPU硬件会在每个像素或每个Quad（2x2像素组）中计算此值。

### 摩尔纹问题的本质与解决

当纹理在屏幕上被大幅缩小时，一个屏幕像素可能覆盖纹理中的多个texel。如果只从Level 0采样，高频细节会被错误地混叠为低频的摩尔纹图案。Mipmap通过预计算的低分辨率版本，确保采样频率满足Nyquist采样定理，从而消除混叠。

### 三线性插值（Trilinear Filtering）

在两个相邻mipmap层级之间进行双线性插值的混合，消除mipmap层级切换时的明显边界。GPU先在两个层级各自做双线性采样，再根据LOD的小数部分在两个结果间线性插值，共需8次texel读取。


## 🛠 工程实践

### Mipmap Chain的生成方式选择

- Box Filter：最简单的生成方式，2x2取平均。速度快但质量一般，可能导致高频细节丢失过快。
- Kaiser Window / Lanczos Filter：更高质量的降采样滤波器，能更好地保留细节和边缘。常用于离线工具（如NVIDIA Texture Tools、Compressonator）生成mipmap。
- 特殊用途的自定义mipmap：例如法线贴图的mipmap需要归一化处理，避免法线在低层级退化；光照贴图的mipmap需要特殊滤波以避免光照泄漏。

### Mipmap Bias调整

- 通过设置正bias值强制使用更高层级（更模糊）的mipmap，可减少远处表面的闪烁和噪点。
- 设置负bias值强制使用更低层级（更清晰）的mipmap，适用于需要保持纹理锐利度的场景（如UI元素、文字纹理）。
- 在OpenGL中通过glTexParameter设置TEXTURE_MIN_LOD/TEXTURE_MAX_LOD或LOD_BIAS；在Vulkan/DirectX中通过Sampler的mipLodBias参数控制。

### Texture Streaming中的Mipmap优先加载策略

- 从最高层级（最小分辨率）开始加载，逐步向Level 0加载，确保纹理始终可用（即使质量较低）。
- 根据相机距离和屏幕空间占用面积计算需要的最高mipmap级别，优先加载可见且重要的纹理层级。
- UE4/5的Texture Streaming系统维护一个Streaming Pool，按优先级调度mipmap级别的加载和卸载。


## ⚠️ 踩坑经验

### Mipmap层级不完整导致的采样错误

如果纹理的mipmap chain不完整（例如只生成了部分层级），GPU在采样时可能访问不存在的层级，导致采样返回黑色（0,0,0）或产生严重的视觉错误。必须确保纹理上传时设置了完整的mipmap层级数。

### Mipmap Bias设置不当

- 正bias过大：远处纹理过度模糊，丢失必要细节，场景看起来雾蒙蒙的、缺乏清晰度。
- 负bias过大：远处纹理出现闪烁和摩尔纹，因为采样频率不足以覆盖高频细节。
- 实践中建议bias值在[-0.5, +1.0]范围内微调，超出此范围通常意味着其他环节存在问题。

### 非2的幂纹理的Mipmap问题

非2的幂（NPOT）纹理在生成mipmap时，某些层级的分辨率可能无法精确减半。虽然现代GPU（OpenGL 2.0+、DirectX 10+）已支持NPOT纹理的mipmap，但在某些移动端GPU上仍可能存在兼容性问题或性能下降。建议在移动端尽量使用2的幂纹理。


## 🔮 延伸思考

### Mipmap Streaming在开放世界中的应用

在开放世界游戏中，场景中可能存在数GB的纹理资源。Mipmap Streaming技术只加载当前视距内所需的mipmap级别，将显存占用控制在预算范围内。关键挑战在于：如何准确预测玩家视线方向以预加载相关纹理、如何处理快速旋转视角时的加载延迟、如何平衡加载带宽与游戏逻辑的IO需求。

### Virtual Texture如何利用Mipmap

虚拟纹理（Virtual Texturing）系统将mipmap的概念推向极致——整个mipmap chain被存储在磁盘上，按需加载可见区域的Page。VT的Page Table本质上就是mipmap的稀疏表示，每个Page对应mipmap chain中特定层级的特定区域。这种机制使得使用超大纹理（如16K甚至更高分辨率的地形贴图）成为可能，而无需将整个纹理加载到显存中。

### Mipmap的现代扩展

Radeon FidelityFX SPD（Single Pass Downsampling）允许在单个Pass中对多个mipmap层级进行自定义降采样，可用于生成特殊的mipmap链（如方差阴影贴图VSM的mipmap）。此外，一些研究探索了各向异性mipmap（Anisotropic Mipmap）和深度感知的mipmap生成方法，以进一步提升采样质量。

---

[← 返回 材质与纹理系统 题目列表](./index.md) | [返回总纲](../00-总纲.md)
