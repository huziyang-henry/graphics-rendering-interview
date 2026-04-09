---
id: Q02.04
title: "什么是BRDF（双向反射分布函数）？请写出Cook-Torrance BRDF的公式并解释每一项的物理含义。"
chapter: 2
chapter_name: "光照与着色模型"
difficulty: intermediate
knowledge_points:
  - id: "02.04"
    name: "BRDF与Cook-Torrance模型"
  - id: "02.05"
    name: "微表面理论（NDF/Fresnel/Geometry）"
tags: ["brdf", "cook-torrance", "ndf", "fresnel", "geometry"]
---

# Q02.04 什么是BRDF（双向反射分布函数）？请写出Cook-Torrance BRDF的公式并解释每一项的物理含义。

**难度：** 🟡 中级
**所属章节：** [光照与着色模型](./index.md)

---

## 🎯 结论

- BRDF（Bidirectional Reflectance Distribution Function，双向反射分布函数）定义了给定入射光方向和出射（观察）方向时，表面反射率的比值关系。
- BRDF的数学定义：f(l
- Cook-Torrance BRDF是PBR中最常用的微表面BRDF模型，公式为：f(l
- 其中D为法线分布函数（NDF），F为菲涅尔项，G为几何遮蔽/阴影项，分母为归一化因子。

## 📐 原理解析

### BRDF的数学定义与物理意义

- BRDF f(l, v)描述了从方向l入射的单位辐照度，有多少比例被反射到方向v。
- 完整的渲染方程：Lo(p, v) = Le(p, v) + ∫_Ω f(l, v) * Li(p, l) * (n · l) dl，其中积分域Ω是上半球。
- BRDF满足两个重要性质：亥姆霍兹互易性（Helmholtz Reciprocity）f(l,v) = f(v,l)，即交换入射和出射方向结果不变；能量守恒，半球积分有界。
- BRDF的单位是sr^-1，因为它是辐射亮度（W/sr/m^2）与辐照度（W/m^2）的比值。

### Cook-Torrance BRDF公式详解

- 完整公式：f(l,v) = (1 - F) * (1 - metallic) * albedo / π + D(h) * F(v,h) * G(l,v,h) / (4 * (n·l) * (n·v))
- 第一项 k_d * f_lambert 是漫反射部分：(1-F)确保能量守恒（被菲涅尔反射的部分不参与漫反射），(1-metallic)确保金属没有漫反射，albedo/π是归一化的Lambert BRDF。
- 第二项是镜面反射部分，由三个物理项和一个归一化因子组成：
-   D(h) - 法线分布函数（Normal Distribution Function）：描述微表面法线等于半程向量h的概率密度。决定了高光的形状和宽度。
-   F(v,h) - 菲涅尔项（Fresnel Term）：描述不同角度下的反射率。决定了掠射角高光增强和金属/非金属的区别。
-   G(l,v,h) - 几何遮蔽项（Geometry Function）：描述微表面之间的相互遮蔽和阴影。决定了粗糙表面的高光衰减。
-   4*(n·l)*(n·v) - 归一化因子：修正微表面BRDF从微表面空间转换到宏观空间时的能量差异。

### 各项的物理直觉

- D项回答「有多少微表面的朝向恰好能将光反射到观察方向」——越多则高光越亮。
- F项回答「这些微表面反射了多少光」——正射时少、掠射时多。
- G项回答「这些微表面是否被其他微表面遮挡」——粗糙表面遮挡严重，高光衰减快。
- 分母4*(n·l)*(n·v)是微表面理论推导出的必然归一化因子，缺少它会导致BRDF值偏大（过亮），尤其在掠射角时问题严重。


## 🛠 工程实践

### GLSL实现Cook-Torrance BRDF

- 典型的GLSL实现结构分为三步：计算NDF、Fresnel和Geometry，然后组合。
- NDF (GGX): D_GGX = a^2 / (pi * ((n·h)^2 * (a^2 - 1) + 1)^2)，其中a = roughness^2。
- Fresnel (Schlick): F_Schlick = F0 + (1.0 - F0) * pow(1.0 - max(h·v, 0.0), 5.0)。
- Geometry (Smith-GGX): G_Smith = G1(n·l) * G1(n·v)，G1 = (n·v) / ((n·v) * (1 - k) + k)，k = (roughness+1)^2/8（直接光照）或k = roughness^2/2（IBL）。
- 最终：specular = (D * F * G) / (4.0 * max(n·l, 0.001) * max(n·v, 0.001))。注意分母的max避免除零。

### HLSL实现注意事项

- HLSL中需要注意向量类型的匹配：float3 vs float4，避免隐式转换导致的精度问题。
- 在Deferred Shading中，BRDF计算在Lighting Pass中执行，N和V从G-Buffer中解码，L从光源位置计算。
- 在Forward+或Clustered Shading中，BRDF计算在片元着色器中执行，需要考虑光源列表的遍历效率。
- 移动端优化：可以将D、F、G的计算合并为共享的中间变量，减少寄存器压力和ALU指令数。


## ⚠️ 踩坑经验

### 分母4*(n·l)*(n·v)容易遗漏

- 这是最常见的实现错误之一。缺少归一化因子会导致BRDF值偏大约4倍（在n·l=n·v=1时），使材质明显过亮。
- 在掠射角时（n·l或n·v接近0），缺少分母会使BRDF值趋向无穷大，产生明显的亮度爆炸。
- 正确做法：分母中使用max(n·l, 0.001)和max(n·v, 0.001)避免除零，同时保持物理正确性。
- 调试技巧：在纯白环境光下渲染一个roughness=0.5的球体，如果中心区域明显过亮（>1.0），很可能是分母遗漏。

### Smith G项的实现细节

- Smith G项有多种实现变体：Smith（原始）、Smith-GGX（Cook-Torrance使用）、Schlick-GGX（Schlick近似）。
- 直接光照和IBL中k值不同：直接光照 k = (roughness+1)^2/8（基于Disney的remapping），IBL k = roughness^2/2（基于Epic的UE4实现）。
- 如果混用两种k值，会导致直接光照和间接光照的高光强度不一致，在混合光照场景中出现明显的亮度跳变。
- G项的另一个常见错误是使用G2（联合遮蔽-阴影函数）代替G1（单方向遮蔽函数）的乘积——Smith模型要求G = G1(l) * G1(v)，而不是直接使用G2(l,v)。

### NDF的roughness参数映射

- GGX NDF中的参数a通常是roughness^2（而非roughness本身），这一映射关系在不同引擎中可能不同。
- Disney的原始论文使用a = roughness^2，UE4也使用这一映射。但某些实现直接使用a = roughness。
- 如果映射不一致，同样的roughness值会产生不同的高光宽度，导致材质在不同引擎间迁移时外观差异明显。
- 建议在项目中明确文档化a和roughness的映射关系，并在材质管线中统一处理。


## 🔮 延伸思考

### BRDF和BTDF的区别

- BRDF（Bidirectional Reflectance Distribution Function）描述反射，BTDF（Bidirectional Transmittance Distribution Function）描述透射。
- BSDF（Bidirectional Scattering Distribution Function）= BRDF + BTDF，是更一般的描述，同时包含反射和透射。
- 对于透明/半透明材质（如玻璃、水），需要同时使用BRDF和BTDF来建模完整的表面行为。
- BTDF在实时渲染中较少使用，因为透明物体的折射需要光线追踪才能精确计算。屏幕空间折射（SSR）是一种近似方案。

### 次表面散射（SSS）如何建模？

- 标准BRDF假设光在入射点附近散射（局部散射），而SSS描述光在材料内部传输一定距离后从不同位置射出（非局部散射）。
- SSS的精确建模需要BSSRDF（Bidirectional Surface Scattering Reflectance Distribution Function），引入了入射点和出射点的距离参数。
- 实时SSS的近似方法包括：
-   - Burley近似（Disney）：用多个高斯核的叠加模拟散射轮廓，计算效率高。
-   - 屏幕空间次表面散射（SSSSS）：在屏幕空间做模糊，受屏幕分辨率和深度不连续性影响。
-   - 传输剖面纹理（Transmission Profile Map）：预计算不同波长的散射距离，在Shader中采样。
- UE5的Subsurface Profile和Unity HDRP的SSS都提供了实时次表面散射的完整解决方案。

---

[← 返回 光照与着色模型 题目列表](./index.md) | [返回总纲](../00-总纲.md)
