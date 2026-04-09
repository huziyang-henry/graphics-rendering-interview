---
id: Q04.04
title: "纹理环绕模式（Wrap Mode）有哪些？在什么场景下应该使用Clamp而不是Repeat？"
chapter: 4
chapter_name: "材质与纹理系统"
difficulty: beginner
knowledge_points:
  - id: "04.03"
    name: "纹理寻址模式（Wrap Mode）"
tags: ["wrap-mode", "clamp", "repeat", "mirror", "uv"]
---

# Q04.04 纹理环绕模式（Wrap Mode）有哪些？在什么场景下应该使用Clamp而不是Repeat？

**难度：** 🟢 初级
**所属章节：** [材质与纹理系统](./index.md)

---

## 🎯 结论

- 四种主要纹理环绕模式：Repeat（重复）、Clamp（钳位）/ Clamp to Edge（钳位到边缘）、Mirror（镜像重复）、Border（边界颜色）。此外还有Mirror Once（半程镜像）等变体。
- Clamp模式适用于需要纹理边缘精确控制的场景，如Lightmap、UI元素、粒子贴图、 decals（贴花）等。当UV坐标不应超出[0,1]范围时使用Clamp，避免纹理边缘出现不自然的重复或接缝。

## 📐 原理解析

### 各模式的数学定义

- Repeat：UV坐标对1取模，即 frac(uv)。UV=1.3等同于UV=0.3，UV=-0.2等同于UV=0.8。纹理在UV空间无限平铺。
- Clamp / Clamp to Edge：UV坐标被限制在[0,1]范围内，即 clamp(uv, 0, 1)。超出范围的UV返回边缘像素的颜色。
- Mirror：UV坐标先对2取模，再根据整数部分的奇偶性决定是否翻转。UV=1.3映射为0.7（翻转），UV=2.3映射为0.3（不翻转）。产生无缝的镜像重复效果。
- Border：超出[0,1]范围的UV返回固定边界颜色（通常为透明黑色或可配置的颜色）。与Clamp to Edge不同，Border不返回边缘像素颜色。
- Mirror Once：UV坐标在[-1,1]范围内镜像，超出1或-1的部分被Clamp。即UV=1.3映射为0.7，UV=-1.3映射为-0.7。

### 硬件实现差异

不同的Wrap Mode在GPU硬件中的实现方式不同。Repeat和Mirror通过简单的位运算或条件判断实现，效率极高。Clamp to Edge需要额外的边界检查，但现代GPU对此有专门优化。Border模式需要额外的颜色寄存器来存储边界颜色。在某些移动端GPU上，非Repeat模式可能引入微小的性能开销。


## 🛠 工程实践

### 各模式的应用场景

- Repeat：地面/墙壁贴图（砖墙、地板纹理的平铺）、UV动画（流动的水面、熔岩效果，通过偏移UV的U或V分量实现）、法线贴图（与Albedo使用相同的Wrap Mode）、环境贴图（Cubemap的面内采样）。
- Clamp to Edge：Lightmap（光照贴图的UV是唯一的，不应重复）、粒子贴图（软粒子需要边缘渐变为透明）、UI元素（按钮、图标的边缘不应重复）、Decal贴花（投影UV超出范围的部分应被裁剪）。
- Mirror：对称纹理的平铺（如壁纸、布料纹理，避免明显的重复感）、某些特殊效果的UV动画。
- Border：需要精确控制UV范围外颜色的场景（如某些后处理效果、自定义的纹理遮罩）。

### UV动画中的Wrap Mode选择

流动效果（如水面、熔岩、能量场）通常使用Repeat模式，通过在顶点着色器或片元着色器中随时间偏移UV坐标实现。关键在于确保纹理本身是无缝可平铺的（Seamless Tiling），否则Repeat模式会暴露接缝。美术工作流中应使用专门的平铺纹理，或在 Substance Designer 中生成无缝纹理。

### Lightmap的特殊处理

Lightmap的UV布局由光照烘焙系统自动生成（通常称为"Lightmap UV"或"UV2"），每个面片在UV空间中有唯一的、不重叠的位置。因此Lightmap必须使用Clamp to Edge模式，避免相邻面片的光照信息互相污染。如果使用Repeat模式，会导致光照泄漏（Light Bleeding）——一个面片的光照信息出现在另一个面片上。


## ⚠️ 踩坑经验

### Clamp to Edge vs Clamp的half-texel偏移问题

在某些图形API中，Clamp模式将UV限制在[0,1]范围，但纹理采样点位于texel中心（0.5/N处）。当UV恰好为0或1时，采样点可能落在纹理边界之外半个texel的位置。Clamp to Edge模式正确处理了这种情况，将采样点钳位到边缘texel的中心。在DirectX 9及更早版本中，这是常见的bug来源；在现代API（DX11+、Vulkan、Metal）中，Clamp通常默认实现为Clamp to Edge。

### Repeat模式在UV Seam处的接缝

当3D模型的UV映射存在接缝（Seam）时，即同一顶点在UV空间中有不同的UV坐标，Repeat模式会在接缝处产生可见的边缘。这是因为接缝两侧的texel来自纹理的不同位置，经过滤波后可能产生颜色不匹配。解决方案包括：在纹理绘制时预留接缝边距（Padding/Dilation）、使用专门的接缝处理工具（如Substance Painter的自动填充功能）。

### 非2的幂纹理的Wrap Mode限制

在旧版OpenGL（2.0之前）和某些移动端GPU上，非2的幂纹理不支持Repeat和Mirror模式，只能使用Clamp。虽然现代API已基本消除此限制，但在做跨平台兼容时仍需注意。如果需要在旧硬件上使用Repeat模式，必须确保纹理尺寸为2的幂。


## 🔮 延伸思考

### 虚拟纹理中的Address Mode处理

在虚拟纹理（Virtual Texturing）系统中，纹理的Address Mode处理更为复杂。VT系统需要在Page Table层面处理UV超出范围的情况，确定应该加载哪个Page。对于Repeat模式，VT需要对UV取模后查找对应的Page；对于Clamp模式，需要将超出范围的请求映射到边缘Page。这种额外的间接寻址可能增加VT系统的复杂度，需要在设计时提前考虑。

### 自定义Wrap Mode的高级应用

在某些特殊效果中，标准Wrap Mode无法满足需求。例如，程序化纹理生成中可能需要基于世界坐标的纹理采样（World Space UV），此时Wrap Mode的选择取决于世界空间坐标的范围。又如在体积渲染（Volume Rendering）中，3D纹理的Wrap Mode需要同时考虑三个维度的边界行为。理解Wrap Mode的本质有助于在这些非标准场景中做出正确的设计决策。

---

[← 返回 材质与纹理系统 题目列表](./index.md) | [返回总纲](../00-总纲.md)
