---
id: Q02.07
title: "什么是IBL（基于图像的光照）？如何用辐照度图和预滤波环境贴图实现IBL？"
chapter: 2
chapter_name: "光照与着色模型"
difficulty: advanced
knowledge_points:
  - id: "02.06"
    name: "IBL（基于图像的光照）"
tags: ["ibl", "irradiance-map", "prefiltered-envmap", "split-sum"]
---

# Q02.07 什么是IBL（基于图像的光照）？如何用辐照度图和预滤波环境贴图实现IBL？

**难度：** 🔵 高级
**所属章节：** [光照与着色模型](./index.md)

---

## 🎯 结论

- IBL（Image-Based Lighting，基于图像的光照）使用环境贴图（HDR全景图）作为光源，通过预计算和实时采样相结合的方式实现高质量的环境光照。
- IBL的核心组件包括三个部分：辐照度图（Irradiance Map）用于Diffuse间接光照、预滤波环境贴图（Prefiltered Environment Map）用于Specular间接光照、BRDF LUT（2D查找表）用于Specular的缩放和偏置。
- Split Sum近似将Specular IBL的复杂积分拆分为两个独立部分的乘积，使实时计算成为可能。
- IBL是PBR管线中不可或缺的组成部分，它让物体在任意环境中都能获得真实感的光照和反射。

## 📐 原理解析

### IBL的基本原理

- 传统光源（方向光、点光源、聚光灯）只能提供直接光照，而真实世界中物体还接收来自四面八方的间接光照。
- IBL的核心思想是用一张HDR全景图（Equirectangular Map或Cubemap）来编码来自所有方向的光照信息。
- 渲染方程中，间接光照的积分可以重写为对环境贴图的卷积：$L_{o,\text{indirect}} = \int f(\mathbf{l}, \mathbf{v}) \cdot \text{env}(\mathbf{l}) \cdot (\mathbf{n} \cdot \mathbf{l}) \, d\mathbf{l}$。
- 这个积分对于每个像素、每个方向都需要计算，直接求解的计算量是 $O(N^2)$（$N$ 为环境贴图分辨率），无法实时完成。
- 因此，IBL通过预计算（离线卷积）将积分结果存储在查找表/贴图中，运行时仅需采样即可。

### 辐照度图（Irradiance Map）

- 辐照度图用于计算间接Diffuse光照。Diffuse BRDF（Lambert）与方向无关（$f_{\text{Lambert}} = \text{albedo} / \pi$），因此卷积简化为对环境贴图的球面平均。
- 卷积公式：

$$ \text{irradiance}(\mathbf{n}) = \frac{1}{\pi} \int_{\Omega} \text{env}(\mathbf{l}) \cdot \max(\mathbf{n} \cdot \mathbf{l}, 0) \, d\mathbf{l} $$

- 物理含义：辐照度图存储了每个法线方向上接收到的总环境光能量，不考虑方向性反射。
- 辐照度图是低频信息（因为球面平均本身就是低通滤波），可以用很小的分辨率（如32x32或64x64 per face的Cubemap）存储。
- 卷积方法：对Cubemap的每个texel，沿法线方向在半球内均匀采样（通常64-128个采样点），计算加权平均。

### 预滤波环境贴图（Prefiltered Environment Map）

- 预滤波环境贴图用于计算间接Specular光照。与Diffuse不同，Specular BRDF（Cook-Torrance）的方向依赖性强，不能简单球面平均。
- 精确卷积需要考虑NDF、Fresnel和Geometry三个项，计算量极大。
- Split Sum近似（Brian Karis, Epic Games 2013）将积分拆分为两部分：
-   Part 1:

$$ \int \text{env}(\mathbf{l}) \cdot D(\mathbf{h}) \cdot G(\mathbf{l}, \mathbf{v}, \mathbf{h}) \cdot \frac{\mathbf{n} \cdot \mathbf{l}}{4 \cdot (\mathbf{n} \cdot \mathbf{v}) \cdot (\mathbf{n} \cdot \mathbf{h})} \, d\mathbf{l} \rightarrow \text{预滤波环境贴图} $$

-   Part 2:

$$ \int F(\mathbf{v}, \mathbf{h}) \cdot G(\mathbf{l}, \mathbf{v}, \mathbf{h}) \cdot \frac{\mathbf{n} \cdot \mathbf{l}}{\mathbf{n} \cdot \mathbf{h}} \, d\mathbf{l} \rightarrow \text{BRDF LUT} $$
- 预滤波环境贴图将不同roughness级别的卷积结果存储在Cubemap的不同mipmap层级中：level 0 = $\text{roughness} = 0$（镜面反射），最高level = $\text{roughness} = 1$（最大模糊）。
- 采样时通过roughness计算LOD级别：`float lod = roughness * (maxMipLevel - 1);` 然后使用textureLod采样。

### BRDF LUT（2D查找表）

- BRDF LUT预计算了Split Sum的Part 2，存储为一张256x256的RG贴图。
- 输入参数：横轴为 $\text{NdotV}$（0-1），纵轴为 $\text{roughness}$（0-1）。
- 输出：R通道为scale（缩放因子），G通道为bias（偏置因子）。
- 最终Specular IBL = $\text{PrefilteredColor} \cdot (F_0 \cdot \text{scale} + \text{bias})$。
- BRDF LUT与场景无关，只需要生成一次即可在所有场景中复用。


## 🛠 工程实践

### Split Sum近似的实现流程

- 步骤1：加载HDR环境贴图（.hdr格式），转换为Cubemap。
- 步骤2：生成辐照度图：对Cubemap进行Diffuse卷积（球面采样，64-128 samples），输出低分辨率Cubemap。
- 步骤3：生成预滤波环境贴图：对Cubemap按roughness级别进行Specular卷积（重要性采样GGX NDF，2048+ samples），输出带mipmap的Cubemap。
- 步骤4：生成BRDF LUT：在Shader中通过蒙特卡洛积分预计算，输出256x256的RG纹理。
- 步骤5：运行时着色：Diffuse = $\text{irradiance} \cdot \text{albedo}$；Specular = $\text{prefilteredColor} \cdot (F_0 \cdot \text{brdfLut.r} + \text{brdfLut.g}) \cdot F_{90} + F_0 \cdot \text{prefilteredColor}$。
- 工具链：HDR环境贴图可从Poly Haven、HDRI Haven等免费资源获取；卷积可使用cmftStudio、IBLBaker等离线工具，或在引擎中实时生成。

### HDR格式处理

- 环境贴图必须使用HDR格式（如RGBE、Float16、Float32），因为真实世界的亮度范围远超LDR（0-255）。
- LDR环境贴图会导致高光区域被截断（clipping），太阳、天空灯等高亮区域丢失信息，反射看起来不自然。
- 常见的HDR格式：.hdr（RGBE，Radiance格式）、.exr（OpenEXR，工业标准）、Float16/Float32纹理。
- Unity和UE都支持直接导入HDR/EXR格式，并自动转换为硬件支持的纹理格式（如ASTC HDR、BC6H）。
- 在移动端，HDR纹理的带宽开销较大，可考虑使用RGBM编码（将HDR值编码到LDR RGBA纹理中）来节省带宽。


## ⚠️ 踩坑经验

### 预滤波贴图的mipmap级别与roughness的映射

- roughness到mipmap level的映射关系需要仔细设置。假设预滤波贴图有N级mipmap（level 0到N-1）：
-   标准映射：$\text{lod} = \text{roughness} \cdot (N - 1)$。例如5级mipmap，$\text{roughness}=0.5 \rightarrow \text{lod}=2.0$。
-   如果mipmap级别不够（如只有4级），roughness较大时会出现精度不足，高光模糊不充分。
-   如果mipmap级别过多（如10+），低roughness时相邻level之间的差异太小，可能产生mipmap banding。
- 经验值：5级mipmap（对应roughness 0, 0.25, 0.5, 0.75, 1.0）对于大多数场景足够。UE4默认使用5级。
- 注意：某些GPU的textureLod在整数LOD时可能产生精度问题，建议使用线性过滤的mipmap采样。

### 采样时的LOD计算

- 在GLSL中采样预滤波贴图时，必须使用textureLod（或textureCubeLod）显式指定LOD级别，不能使用自动mipmap选择。
- 自动mipmap选择基于屏幕空间导数，会导致相邻像素使用不同的LOD级别，产生锯齿和闪烁。
- 正确做法：`float lod = roughness * float(maxMipLevel); vec3 prefilteredColor = textureLod(prefilteredMap, R, lod).rgb;`
- 在某些移动端GPU上，textureLod可能不支持非整数LOD，需要使用textureGrad或手动在两个整数LOD之间插值。

### 环境贴图的接缝问题

- Equirectangular到Cubemap的转换可能在面的边界产生接缝（尤其是极点区域）。
- 低分辨率辐照度图由于采样密度低，接缝问题可能更明显。
- 解决方案：在Cubemap的每个面边缘添加额外的padding（1-2像素），使用边缘像素填充；或在采样时对UV进行clamp。
- 预滤波贴图在低mipmap级别时，面的边界可能产生不连续。使用GPU原生的Cubemap采样（而非手动计算面索引）可以自动处理边缘插值。


## 🔮 延伸思考

### 动态IBL的挑战

- 标准IBL假设环境是静态的（预计算的贴图不变），但很多场景需要动态更新环境光照：
-   - 昼夜循环：天空颜色和光照强度随时间变化。
-   - 动态物体：大型移动物体（如门、车辆）会显著改变局部环境。
-   - 破坏效果：墙壁被炸毁后，室内环境光照需要重新计算。
- 动态更新预滤波贴图的计算量很大（每个roughness级别都需要卷积），无法每帧重新生成。
- 可能的解决方案：预计算多组环境贴图并在之间插值（适用于昼夜循环）；使用屏幕空间反射（SSR）补充动态反射；降低预滤波贴图的分辨率和采样数以加速更新。

### 实时的环境贴图更新策略

- UE5的Lumen使用动态全局光照，结合软件光追和SDF（Signed Distance Field）来实时计算间接光照，部分替代了传统IBL。
- 屏幕空间方法（SSR、SSAO、SSGI）可以在一定程度上提供动态的间接光照，但受限于屏幕空间信息（无法反映屏幕外物体的反射）。
- 混合方案：静态环境使用预计算IBL，动态物体使用SSR/SSGI补充，两者通过权重混合。
- 未来方向：硬件光线追踪（RTX）使得实时的环境贴图生成成为可能——每帧从相机位置发射少量光线，实时更新Cubemap。NVIDIA的RTXGI和AMD的FidelityFX SSGI都在探索这一方向。

---

[← 返回 光照与着色模型 题目列表](./index.md) | [返回总纲](../00-总纲.md)
