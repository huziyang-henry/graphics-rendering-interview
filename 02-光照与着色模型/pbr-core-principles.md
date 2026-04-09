---
id: Q02.03
title: '请解释PBR（基于物理的渲染）的核心原则。为什么说PBR比传统光照模型更"物理正确"？'
chapter: 2
chapter_name: "光照与着色模型"
difficulty: intermediate
knowledge_points:
  - id: "02.03"
    name: "PBR核心原则与能量守恒"
tags: ["pbr", "energy-conservation", "microfacet"]
---

# Q02.03 请解释PBR（基于物理的渲染）的核心原则。为什么说PBR比传统光照模型更"物理正确"？

**难度：** 🟡 中级
**所属章节：** [光照与着色模型](./index.md)

---

## 🎯 结论

- PBR（Physically Based Rendering）的三大核心原则是：微表面理论（Microfacet Theory）、能量守恒（Energy Conservation）和菲涅尔效应（Fresnel Effect）。
- 微表面理论将宏观表面建模为由无数微小镜面组成的集合，每个微表面的朝向由法线分布函数（NDF）描述。
- 能量守恒确保反射光和折射光的能量之和不超过入射光能量，避免了传统模型中「过亮」的问题。
- 菲涅尔效应描述了掠射角反射率增大的物理现象，是区分金属和非金属材质的关键特征。
- PBR比传统模型更「物理正确」，因为它基于真实世界的光学测量数据（如MERL BRDF数据库），并且在任何光照条件下都能产生一致、可预测的结果。

## 📐 原理解析

### 微表面理论（Microfacet Theory）

- 微表面理论假设：宏观上看似光滑的表面，在微观尺度上由无数微小平面（microfacet）组成，每个微表面都是一个完美的镜面反射器。
- 每个微表面的法线方向不同，由法线分布函数（Normal Distribution Function, NDF）D(h)描述其统计分布，h为微表面法线方向。
- 只有那些法线方向恰好等于半程向量h = normalize(l + v)的微表面，才能将光线l反射到视线方向v。
- 表面粗糙度（roughness）决定了微表面法线的分散程度：roughness=0表示所有微表面法线一致（完美镜面），roughness=1表示法线完全随机分布（理想漫反射面）。
- 微表面理论统一了漫反射和镜面反射的描述框架，是PBR的理论基石。

### 能量守恒（Energy Conservation）

- 能量守恒定律要求：表面反射和透射的总能量不能超过入射光的能量。
- 在数学表达上，BRDF f(l,v)对所有入射方向的半球积分必须小于等于1/π（对于反射BRDF）。
- 在PBR实践中，能量守恒体现在：Diffuse和Specular共享入射能量，菲涅尔效应决定了能量在两者之间的分配。
- 具体公式：k_d + k_s <= 1，其中k_d是Diffuse比例，k_s是Specular比例。对于金属，k_d=0（所有能量被反射）；对于非金属，k_d = 1 - F（F为菲涅尔反射率）。
- 传统Phong模型不满足能量守恒：spec = (R·V)^n 的半球积分远大于1，导致高光区域过亮，尤其在低roughness时问题严重。

### 菲涅尔效应（Fresnel Effect）

- 菲涅尔效应描述了一个重要的物理现象：当观察角度从正射变为掠射时，所有材质的反射率都会增大，在掠射角（90度）时趋近于100%。
- 日常生活中可以观察到：水面正看时透明，掠射看时像镜子；树叶正看时绿色，掠射看时泛白光。
- 菲涅尔效应的精确公式涉及复数折射率的偏振光计算（Fresnel方程），工程中常用Schlick近似：F(θ) = F0 + (1 - F0)(1 - cosθ)^5。
- F0是正射时的反射率：非金属（绝缘体）F0约0.02-0.08（通常取0.04），金属（导体）F0约0.5-1.0且与波长相关（有色反射）。
- 菲涅尔效应是PBR区分金属和非金属的核心机制，也是传统光照模型缺失的关键物理现象。


## 🛠 工程实践

### Metallic-Roughness工作流

- Metallic-Roughness是当前最主流的PBR工作流，由Disney于2012年提出，被UE4/Unity/Blender等广泛采用。
- 核心参数：Base Color（反照率）、Metallic（金属度，0或1）、Roughness（粗糙度，0-1）、Normal Map（法线贴图）、AO（环境光遮蔽）。
- Metallic贴图通常为二值化（0或1），中间值仅用于过渡区域（如金属上的灰尘、锈蚀）。
- Roughness控制高光宽度和环境反射的模糊程度：0=完美镜面，1=完全漫反射。
- 优势：参数直觉性强、材质定义明确（金属vs非金属）、跨引擎兼容性好（glTF 2.0标准格式）。

### Specular-Glossiness工作流

- Specular-Glossiness是另一种PBR工作流，核心参数为：Diffuse Color、Specular Color（F0颜色）、Glossiness（光泽度 = 1 - roughness）。
- 优势：可以更精确地控制F0值（不限于0.04和albedo两个极端），适合需要特殊反射率的材质（如宝石、某些塑料）。
- 劣势：参数空间更大，更容易调出非物理的结果（如同时设置高Diffuse和高Specular）；不如Metallic-Roughness直觉。
- Substance Painter/Designer同时支持两种工作流，可以互相转换。UE4从4.26版本开始推荐Metallic-Roughness。

### PBR材质验证工具

- Substance Painter的PBR Validator可以检测材质是否符合物理约束（如金属的Diffuse应为黑色、非金属的Specular应在0.02-0.08范围）。
- UE4的Material Editor中可以通过Debug View（如Shading Complexity、Quad Overdraw）验证PBR材质的正确性。
- glTF Validator可以检查导出的glTF 2.0文件中PBR金属度贴图和粗糙度贴图的值域是否合法。


## ⚠️ 踩坑经验

### roughness不能为0

- roughness=0意味着完美镜面反射，但在实时渲染中会导致问题：GGX NDF在roughness=0时退化为Dirac delta函数，高光变成一个无限小的亮点。
- 在屏幕空间中，这个亮点可能落在像素之间而完全不可见，或者因为数值精度问题产生闪烁。
- 工程实践：通常将roughness的最小值钳制为0.04~0.045，既保持高度光滑的外观又避免数值问题。
- 在预滤波环境贴图的mipmap采样中，roughness=0对应mipmap level 0（原始贴图），此时采样结果取决于单个像素，容易产生噪点。

### metallic的过渡处理

- Metallic贴图在金属和非金属交界处不应是硬边（0/1跳变），否则菲涅尔效应会在边界处产生明显的不连续。
- 过渡区域（如金属表面的灰尘、氧化层、油漆）的metallic值应在0-1之间平滑过渡。
- 但过渡区域的宽度需要控制——过宽的过渡会导致材质看起来「脏」或不确定。
- 经验法则：过渡区域通常不超过2-3像素宽，在纹理分辨率足够高时（2048+）这不是问题。

### Base Color的值域限制

- PBR要求Base Color的亮度在合理范围内：非金属的Luminance应在50-240 sRGB之间，金属的Luminance应在180-255 sRGB之间。
- 纯黑（0,0,0）的Base Color在物理上不存在——即使最暗的炭黑也有约0.02-0.04的反照率。
- 纯白（255,255,255）的Base Color意味着100%反射率，违反能量守恒（没有光被吸收转化为热量）。
- Substance Painter的PBR Validator会将超出范围的Base Color标记为警告。


## 🔮 延伸思考

### PBR是否是最终答案？

- PBR是当前实时渲染的最佳实践，但并非万能。以下物理现象超出了标准PBR的覆盖范围：
- 次表面散射（Subsurface Scattering）：光在半透明材料（皮肤、玉石、蜡烛）内部的传输。标准PBR的Lambert Diffuse假设光在入射点附近散射，无法模拟SSS的深度透光效果。
- 薄膜干涉（Thin-film Interference）：如肥皂泡、油膜上的彩虹色，由多层薄膜的光程差引起。
- 各向异性反射（Anisotropic Reflection）：如拉丝金属、头发、丝绸，反射率依赖于方向。
- 双折射（Birefringence）：如水晶、云母，不同偏振方向的光有不同的折射率。
- 荧光/磷光（Fluorescence/Phosphorescence）：吸收短波长光后发射长波长光。
- 这些现象需要扩展的PBR模型（如Disney Principled BRDF的Sheen/Clear Coat/Thin Film扩展）或专门的渲染技术来处理。

### PBR的未来发展方向

- 路径追踪（Path Tracing）在实时渲染中的普及（UE5的Lumen、Unity的HDRP路径追踪）正在模糊离线和实时的界限。
- 基于神经网络的材质表示（如Neural BRDF、MaterialGAN）可能成为下一代材质建模的方向。
-  spectral rendering（光谱渲染）用完整的光谱分布替代RGB三通道，可以更准确地模拟色散、荧光等波长相关现象。
- PBR的核心理念——基于物理测量、参数化材质描述、跨光照条件的一致性——将继续指导未来的渲染技术发展。

---

[← 返回 光照与着色模型 题目列表](./index.md) | [返回总纲](../00-总纲.md)
