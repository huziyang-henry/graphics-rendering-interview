---
id: Q02.02
title: "什么是环境光（Ambient）、漫反射（Diffuse）和镜面反射（Specular）？它们各自的物理直觉是什么？"
chapter: 2
chapter_name: "光照与着色模型"
difficulty: beginner
knowledge_points:
  - id: "02.02"
    name: "光照分量（环境光/漫反射/镜面反射）"
tags: ["ambient", "diffuse", "specular", "lighting"]
---

# Q02.02 什么是环境光（Ambient）、漫反射（Diffuse）和镜面反射（Specular）？它们各自的物理直觉是什么？

**难度：** 🟢 初级
**所属章节：** [光照与着色模型](./index.md)

---

## 🎯 结论

- 环境光（Ambient）、漫反射（Diffuse）和镜面反射（Specular）是经典光照模型的三个核心分量，分别模拟间接光照、表面散射和镜面反射三种不同的光与物质交互过程。
- Ambient是对全局间接光照的简化近似，假设来自所有方向的均匀光照。
- Diffuse遵循Lambert余弦定律，描述光进入表面后在材料内部发生多次散射、最终以随机方向重新射出的现象。
- Specular描述光在表面微观结构上的直接反射，产生高光效果，反射方向与入射方向关于法线对称。

## 📐 原理解析

### 环境光（Ambient）的物理直觉

- 真实世界中，物体不仅接收来自光源的直接照射，还接收来自周围环境的间接光照（其他物体表面的反射光、天空散射光等）。
- 精确计算间接光照需要全局光照算法（光线追踪、辐射度等），计算量极大。
- Ambient是对这一复杂过程的极简近似：假设从所有方向均匀入射的常量光照，计算公式为 I_ambient = k_a * I_a，其中k_a为环境光系数，I_a为环境光强度。
- 物理直觉：Ambient模拟的是光在场景中的「漫反射弹射」——光从物体A反射到物体B，再从B反射到C……经过多次弹射后趋于均匀分布。
- 更高级的环境光近似包括球谐函数（Spherical Harmonics, SH）光照、环境光遮蔽（Ambient Occlusion, AO）等。

### 漫反射（Diffuse）的物理直觉

- 当光照射到非金属粗糙表面时，大部分光能进入材料内部，在微观颗粒之间经历多次散射（吸收和重新发射），最终以近似均匀的方向分布从入射点附近射出。
- 这一过程遵循Lambert余弦定律：表面接收的光能量与入射角余弦成正比。公式为 I_diffuse = k_d * I_light * max(N · L, 0)。
- Lambert模型的物理含义：单位面积上接收到的光通量随入射角增大而减小——倾斜照射时，同样的光束覆盖更大的表面积。
- 漫反射光的颜色由材料本身的吸收特性决定：白光照射红色物体时，绿色和蓝色分量被吸收，只有红色分量被散射出来。
- 漫反射光的偏振态是随机的（因为多次散射打乱了光的偏振方向），这也是区分漫反射和镜面反射的方法之一。

### 镜面反射（Specular）的物理直觉

- 镜面反射发生在材料表面（而非内部），光在表面微观平整区域发生「镜面」反射，反射角等于入射角。
- 对于理想镜面，所有光沿反射方向R射出；对于粗糙表面，微表面的朝向随机分布，高光在R附近散开。
- Specular不改变光的颜色（对非金属而言），因为反射发生在表面，光没有进入材料内部被吸收。
- 高光的形状和强度取决于表面粗糙度：越光滑的表面高光越集中越亮，越粗糙的表面高光越分散越柔和。


## 🛠 工程实践

### Unity中的三分量Shader实现

- Unity的Legacy Shader（如Diffuse、Specular、Bumped Specular）直接暴露Ambient、Diffuse、Specular三个参数供美术调整。
- Ambient分量通常由Lighting Settings中的Source（Skybox/Gradient/Color）和Ambient Intensity控制。
- 在Unity的Surface Shader中，可以通过o.Albedo（Diffuse颜色）、o.Specular（Specular颜色）、o.Gloss（高光锐度）分别控制。
- Unity的Lightweight Render Pipeline（LWRP）/ Universal Render Pipeline（URP）中，环境光由GI系统提供，通过采样Lighting Probe或Lightmap获取。

### Unreal Engine中的实现

- UE4/UE5的Lit材质中，Ambient对应Base Color在间接光照下的贡献，通过Diffuse Color间接光照通道计算。
- UE使用球谐函数（SH）的3阶近似（L2，共9个系数）来编码环境光的方向分布，比均匀Ambient更精确。
- Specular在UE中由Specular输入引脚控制（在Metallic-Roughness工作流中由Roughness间接控制高光宽度）。
- UE的Screen Space Reflections（SSR）为Specular分量提供实时反射效果，补充了IBL在动态场景中的不足。

### 球谐函数（Spherical Harmonics）环境光表示

- SH是一种在球面上定义的正交基函数，类似于傅里叶级数在周期函数中的作用。
- 3阶SH（L2）使用9个系数（3个L0 + 3个L1 + 5个L2）即可编码低频环境光信息，存储开销极小。
- Unity的Lighting Probes和UE的Lightmass都使用SH来存储间接光照信息。
- SH的主要局限是无法表示高频细节（如尖锐的反射、点光源），因此只适合Ambient/Diffuse级别的近似。


## ⚠️ 踩坑经验

### 纯Ambient导致物体看起来扁平

- 如果只使用常量Ambient而不加任何方向性光照，物体将缺乏明暗变化，看起来像一个均匀着色的平面图形。
- 这是最常见的初学者错误之一——场景中只有Ambient Light，没有Directional Light或其他光源。
- 解决方案：至少添加一个方向光来提供明暗对比；使用Hemisphere Ambient（天空色+地面色）代替纯色Ambient来增加方向感。
- 在户外场景中，Hemisphere Light（上蓝下棕）可以低成本地模拟天空散射光和地面反射光的差异。

### Diffuse能量不守恒问题

- 经典的Lambert Diffuse公式 I = k_d * (N · L) 本身是能量守恒的（半球积分等于pi，需要除以pi归一化）。
- 但在实际Shader中，很多实现省略了1/pi的归一化因子，导致Diffuse反射的能量偏大。
- 在PBR中，正确的Lambert Diffuse应为 f_diffuse = albedo / pi，这个1/pi因子确保了BRDF的积分守恒。
- 如果从传统管线迁移到PBR管线，忘记添加1/pi会导致整体亮度偏高约3.14倍（约+5 EV），需要重新校准所有材质参数。

### Ambient和Diffuse的混淆

- 有些开发者误以为Ambient就是「暗部的Diffuse」，这是不准确的。Ambient是间接光照的近似，Diffuse是直接光照的散射分量。
- 在PBR框架下，Ambient的概念被IBL（基于图像的光照）替代——间接Diffuse通过辐照度图（Irradiance Map）采样，间接Specular通过预滤波环境贴图采样。
- 在光照烘焙（Lightmapping）中，间接光照被预计算并存储在Lightmap中，此时Ambient项通常设为黑色或极低值，避免重复计算。


## 🔮 延伸思考

### PBR如何重新定义这三个概念？

- 在PBR中，Ambient不再是一个独立的常量，而是通过IBL系统精确计算：间接Diffuse使用辐照度图（Irradiance Map），间接Specular使用预滤波环境贴图（Prefiltered Environment Map）。
- Diffuse在PBR中被重新建模为能量守恒的Lambert模型（除以pi），并且与Specular通过菲涅尔效应（Fresnel）进行能量分配：k_d = (1 - F) * (1 - metallic)。
- Specular在PBR中由Cook-Torrance微表面BRDF精确描述，包含法线分布函数（NDF）、菲涅尔项（F）和几何遮蔽项（G）三个物理量。
- 传统三分量的简单叠加被替换为物理正确的积分方程：Lo = Le + ∫[f(l,v) * Li(l) * (n·l)] dl，其中f(l,v)就是BRDF。

### Split Sum近似如何处理环境光的Specular分量？

- Split Sum近似由Brian Karis在Epic Games的PBR实现中提出，将Specular IBL的积分拆分为两个独立部分的乘积：预滤波环境贴图（Prefiltered Env Map）和BRDF LUT（2D查找表）。
- 预滤波环境贴图对不同roughness级别进行卷积，存储在不同mipmap层级中，通过roughness控制LOD采样。
- BRDF LUT预计算了菲涅尔和几何遮蔽项的积分结果，输入为roughness和NdotV，输出为缩放（scale）和偏置（bias）两个值。
- 最终Specular IBL = PrefilteredColor * (F0 * scale + bias)，在保证视觉质量的同时将实时计算量降到可接受范围。

---

[← 返回 光照与着色模型 题目列表](./index.md) | [返回总纲](../00-总纲.md)
