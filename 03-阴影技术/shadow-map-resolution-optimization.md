---
id: Q03.04
title: "Shadow Map的分辨率不足时会出现什么问题？除了提高分辨率，还有哪些方法可以改善阴影质量？"
chapter: 3
chapter_name: "阴影技术"
difficulty: intermediate
knowledge_points:
  - id: "03.01"
    name: "Shadow Map基础原理"
  - id: "03.02"
    name: "软阴影技术（PCF/PCSS/VSM/ESM）"
tags: ["shadow-map", "resolution", "aliasing", "lisp-sm"]
---

# Q03.04 Shadow Map的分辨率不足时会出现什么问题？除了提高分辨率，还有哪些方法可以改善阴影质量？

**难度：** 🟡 中级
**所属章节：** [阴影技术](./index.md)

---

## 🎯 结论

- Shadow Map分辨率不足是实时阴影质量下降的最主要原因，表现为锯齿状阴影边缘（Staircase Artifacts）、漏光（Light Leaking）和透视锯齿（Perspective Aliasing）等问题。
- 除了直接提高Shadow Map分辨率（这会增加内存和带宽开销），还可以通过CSM分区、可滤波阴影技术（VSM/ESM）、自适应级联划分（SDSM）、透视阴影映射（LiSPSM）等多种方法来改善阴影质量。

## 📐 原理解析

### 分辨率不足的核心问题

- 锯齿状阴影边缘（Staircase Artifacts）：当Shadow Map的一个纹素覆盖了屏幕上的多个像素时，多个屏幕像素会查询到相同的Shadow Map深度值，导致阴影边缘呈现阶梯状。这在Shadow Map的纹素与屏幕像素的映射比例较大时尤为明显。
- 漏光（Light Leaking）：当Shadow Map分辨率不足以精确表示遮挡物的几何细节时，狭窄的缝隙或细小的遮挡物可能在Shadow Map中“丢失”，导致本应被遮挡的区域出现不正确的光照。例如，墙壁之间的窄缝在低分辨率Shadow Map中可能完全不可见。
- 透视锯齿（Perspective Aliasing）：由于透视投影的特性，近处物体在屏幕上占据更多像素，但在光源空间的Shadow Map中却不一定获得相应的纹素密度。近处需要的Shadow Map精度远高于远处，但均匀分辨率的Shadow Map无法满足这种非均匀需求。

### Perspective Aliasing的数学分析

Perspective Aliasing的严重程度可以用Shadow Map纹素在世界空间中的投影面积来衡量。对于一个距离摄像机d处的表面，其Shadow Map纹素对应的世界空间面积与d^2成正比（透视缩放），但与光源到该表面的距离关系取决于光源类型。方向光的Shadow Map纹素在世界空间中的大小是恒定的，因此近处表面的Perspective Aliasing最为严重。


## 🛠 工程实践

### CSM——分区提高有效分辨率

CSM通过将视锥体分割为多个级联，每个级联使用独立的Shadow Map，相当于为不同深度范围分配了不同的有效分辨率。近处级联的Shadow Map覆盖范围小，等效分辨率高；远处级联覆盖范围大，等效分辨率低。这是目前工业界解决Perspective Aliasing最主流的方案。

### VSM/ESM——可滤波阴影技术

- Variance Shadow Map（VSM）和Exponential Shadow Map（ESM）允许对Shadow Map进行预滤波（如Mipmap），从而在较低分辨率下实现大范围的软阴影效果。虽然它们不能直接提高硬阴影的分辨率，但通过模糊处理可以有效掩盖锯齿。
- VSM的Mipmap可以在GPU硬件层面高效生成，ESM的指数表示也支持双线性滤波。这意味着即使Shadow Map分辨率较低，经过适当的滤波后也能获得视觉上可接受的阴影质量。

### SDSM——自适应级联划分

Sample Distribution Shadow Maps（SDSM）通过分析前一帧的深度缓冲来动态确定级联的分割位置。它统计深度缓冲中实际存在的深度分布，将级联边界设置在场景内容密集的区域，避免将Shadow Map分辨率浪费在空旷区域。SDSM特别适合深度分布不均匀的场景（如室内外过渡、垂直方向上有高度变化的场景）。

### LiSPSM——透视阴影映射

Light Space Perspective Shadow Maps（LiSPSM）在光源空间中引入一个透视变换，使得近处的Shadow Map纹素密度更高。与CSM不同，LiSPSM使用单张Shadow Map，通过扭曲光源空间来实现精度重分配。LiSPSM可以与CSM结合使用，在每个级联内部进一步优化精度分布。


## ⚠️ 踩坑经验

### VSM的光晕（Light Bleeding）问题

VSM在低分辨率下使用Mipmap进行大范围模糊时，光晕问题会显著加剧。当遮挡物和接收面之间的深度差较大时，Chebyshev不等式的上界过于宽松，导致本应完全处于阴影中的区域出现部分光照。这是VSM最严重的缺陷之一，通常需要通过提高分辨率或使用更高阶的矩（如Moment Shadow Maps）来缓解。

### 提高分辨率对带宽的影响

- Shadow Map分辨率翻倍意味着内存占用翻4倍（2D），带宽开销也相应增加。在移动端，带宽是稀缺资源，过高的Shadow Map分辨率可能导致内存带宽瓶颈。
- 实践中需要权衡阴影质量与性能预算。一种常用的策略是使用动态分辨率——根据GPU负载自动调整Shadow Map分辨率，在性能紧张时降低分辨率以保证帧率稳定。
- 使用深度纹理格式（如GL_DEPTH_COMPONENT16）而非RGBA格式可以减少Shadow Map的内存占用。


## 🔮 延伸思考

### 虚拟阴影贴图（VSM）的根本性解决思路

UE5的Virtual Shadow Maps从根本上重新思考了分辨率分配问题。它借鉴虚拟纹理（Virtual Texturing）的思路，将Shadow Map划分为固定大小的Page（如128x128），按屏幕像素的需求动态加载和缓存这些Page。屏幕上每个像素都请求自己所需的Shadow Map Page，只有被请求的Page才会被渲染和缓存。这意味着Shadow Map的分辨率可以非常高（如16384x16384甚至更高），但实际使用的内存只与屏幕上的可见内容相关，实现了分辨率按需分配。

---

[← 返回 阴影技术 题目列表](./index.md) | [返回总纲](../00-总纲.md)
