---
id: Q05.02
title: "Tone Mapping的作用是什么？ACES曲线和Reinhard曲线各有什么特点？为什么HDR渲染必须做Tone Mapping？"
chapter: 5
chapter_name: "后处理效果"
difficulty: intermediate
knowledge_points:
  - id: "05.01"
    name: "Bloom与Tone Mapping"
tags: ["tone-mapping", "aces", "reinhard", "hdr", "exposure"]
---

# Q05.02 Tone Mapping的作用是什么？ACES曲线和Reinhard曲线各有什么特点？为什么HDR渲染必须做Tone Mapping？

**难度：** 🟡 中级
**所属章节：** [后处理效果](./index.md)

---

## 🎯 结论

- Tone Mapping将HDR渲染产生的高动态范围亮度值映射到显示器可显示的低动态范围（LDR），同时尽可能保持画面的对比度、色彩饱和度和视觉真实感。它是连接HDR渲染管线与最终显示输出的必经桥梁。

## 📐 原理解析

### 为什么HDR渲染必须做Tone Mapping

- HDR渲染过程中，光照计算产生的颜色值可以远超[0,1]范围。例如，太阳表面的亮度可能达到数百甚至数千，金属高光反射也可能超过10.0
- 显示器（无论是sDR还是HDR显示器）都有其最大亮度限制。sDR显示器的标准白点为1.0（约100 nits），即使HDR显示器也有其峰值亮度上限
- 如果直接将HDR值clamp到[0,1]，会导致所有超过1.0的亮度变成纯白，丢失大量高光细节和层次感
- Tone Mapping的核心目标是在压缩亮度范围的同时，保持视觉上的对比度感知和色彩保真度

### Reinhard曲线

- 公式：L_out = L_in / (1 + L_in)，这是一个简单的全局映射函数
- 特点：实现简单，计算开销极低；对所有亮度值进行统一的压缩，没有区分阴影、中间调和高光的处理
- 主要缺陷：高光区域被压缩得过于平坦，容易产生\
- （Washed-out）的感觉。因为随着输入亮度增大，输出趋近于1.0的速度太快，高光缺乏\
- （Shoulder）过渡
- Reinhard的改进版本（Extended Reinhard）通过引入白点参数L_white来控制压缩程度：L_out = L_in * (1 + L_in/L_white^2) / (1 + L_in)

### ACES曲线（Academy Color Encoding System）

- ACES是电影与电视工程师学会（AMPAS）制定的颜色编码标准，其Tone Mapping曲线被广泛用于游戏和影视行业
- ACES曲线的核心优势在于其精心设计的S形曲线（S-Curve）：在阴影区域有较陡的上升（保留暗部细节），中间调区域接近线性（保持自然感），高光区域有平缓的肩部过渡（高光渐变到白色而非突然截断）
- ACES拟合公式（Stephen Hill的简化版本）：使用有理多项式近似，避免了分段函数的实现复杂度
- 相比Reinhard，ACES在高光区域的表现明显更好——高光不会突然变成纯白，而是有一个自然的渐变过渡，保留了高光区域的色彩和层次


## 🛠 工程实践

### ACES拟合函数的实现

- 常用的ACES拟合实现来自Stephen Hill的博客，使用有理多项式近似原始ACES矩阵变换
- 实现时需要注意：ACES曲线应该只应用于亮度通道（Luminance），然后按比例缩放RGB分量，以避免色彩偏移
- 也可以直接对RGB三个通道分别应用ACES曲线，但需要注意这可能导致高饱和度颜色的色相偏移
- UE4/UE5默认使用ACES Tone Mapping，Unity的HDRP也提供ACES选项

### Tone Mapping在管线中的位置

- 标准顺序：Bloom → Tone Mapping → Gamma Correction → Color Grading。Tone Mapping必须在Bloom之后，因为Bloom产生的叠加值也是HDR值，需要一起被映射
- Tone Mapping必须在Gamma Correction之前，因为Tone Mapping假设输入为线性空间的HDR值
- Color Grading（LUT查找）通常在Tone Mapping之后进行，此时颜色值已经在[0,1]范围内，便于LUT采样

### 曝光补偿（Exposure Compensation）

- 在Tone Mapping之前乘以曝光值：color *= exposure，可以整体调整画面的明暗
- 曝光补偿通常提供给美术人员作为全局参数，用于快速调整场景的整体亮度感觉
- 自动曝光（Auto Exposure）：通过计算当前帧的平均/中值亮度，自动调整曝光值以适应不同亮度的场景。实现时通常使用直方图统计，并加入时间平滑以避免曝光跳变


## ⚠️ 踩坑经验

### Tone Mapping前的颜色空间必须是线性空间

- 如果输入颜色不在线性空间（例如已经做了Gamma校正），Tone Mapping的结果会严重失真——中间调会过亮，高光压缩不正确
- 常见错误：在sRGB纹理直接作为渲染输入时，忘记在采样时进行sRGB→Linear转换
- 验证方法：检查Tone Mapping前的中间RT格式是否为FLOAT类型（如R11G11B10_FLOAT或RGBA16_FLOAT），确保没有在之前的Pass中意外做了Gamma编码

### ACES曲线在极高亮度下的行为

- ACES曲线在输入亮度极高时（>1000），输出会轻微超过1.0然后回落，导致\
- 效果（亮度越高输出反而略低）
- 虽然这种情况在实际渲染中很少出现，但在处理太阳、爆炸等极端高亮光源时需要注意
- 解决方案：在应用ACES之前，先对极端值进行clamp或使用改进版的ACES曲线（如ACEScc的变体）

### 移动端Tone Mapping的精度问题

- 部分移动端GPU对半精度浮点（FP16）的支持不完善，可能导致Tone Mapping计算精度不足
- 精度不足的表现为色带（Banding）——在平滑的渐变区域出现可见的阶梯状色阶
- 解决方案：使用R11G11B10_FLOAT格式（无Alpha通道但精度足够）或在Tone Mapping后添加微量Dithering（抖动）来掩盖色带


## 🔮 延伸思考

### HDR显示器的Tone Mapping

- 在HDR10/Dolby Vision显示器上，显示器的峰值亮度可达1000~10000 nits，远超sDR的100 nits
- 此时Tone Mapping的目标从\
- 变为\
- ，使用PQ（Perceptual Quantizer）曲线或ScRGB颜色空间
- PQ曲线（ST 2084）基于人眼的亮度感知模型，在整亮度范围内提供感知均匀的量化
- 游戏需要同时支持sDR和HDR输出，通常需要两套Tone Mapping参数或一个自适应的映射方案

### Adaptive Tone Mapping（自适应色调映射）

- 自适应Tone Mapping根据场景内容动态调整映射曲线，例如在暗场景中保留更多暗部细节，在亮场景中保留更多高光细节
- 电影行业常用的Filmic Tone Mapping本质上就是一种自适应方案，它模拟了胶片对不同亮度区域的非线性响应
- 更高级的方案包括基于直方图的Tone Mapping（分析整个画面的亮度分布来优化映射曲线）和基于视觉注意力模型的Tone Mapping（对用户关注的区域保留更多动态范围）

---

[← 返回 后处理效果 题目列表](./index.md) | [返回总纲](../00-总纲.md)
