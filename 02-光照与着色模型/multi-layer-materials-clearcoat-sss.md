---
id: Q02.09
title: "在PBR框架下，如何正确处理多层材质（如清漆层Coat、次表面散射SSS）？"
chapter: 2
chapter_name: "光照与着色模型"
difficulty: expert
knowledge_points:
  - id: "02.08"
    name: "多层材质与高级着色"
  - id: "02.04"
    name: "BRDF与Cook-Torrance模型"
tags: ["clearcoat", "sss", "multi-layer", "disney-brdf"]
---

# Q02.09 在PBR框架下，如何正确处理多层材质（如清漆层Coat、次表面散射SSS）？

**难度：** 🟣 专家级
**所属章节：** [光照与着色模型](./index.md)

---

## 🎯 结论

- 多层材质通过分层BRDF叠加实现，每层有独立的粗糙度、法线分布函数（NDF）、菲涅尔效应和几何衰减项。
- 常见的多层材质模型包括：Clear Coat（清漆层，如汽车漆面、钢琴漆）、次表面散射（SSS，如皮肤、玉石）、Sheen（绒毛光泽，如布料）。
- 多层BRDF的核心挑战是能量守恒——简单的线性叠加会导致总反射率超过100%，必须使用物理正确的串行或并行混合策略。
- Split Sum近似在多层材质中同样适用，但需要为每层独立预计算预滤波贴图或使用统一的roughness参数。

## 📐 原理解析

### Clear Coat模型

- Clear Coat（清漆层）模拟在基础材质上覆盖一层透明介质（如清漆、蜡、水膜）的效果。
- 典型应用：汽车漆面（透明清漆 + 金属底色）、钢琴漆、涂漆木材、湿润表面。
- 物理模型：顶层为透明介质层（F0≈0.04，独立roughness），底层为标准PBR材质。
- 光与Clear Coat的交互：光首先到达顶层界面，一部分被菲涅尔反射（Clear Coat Specular），其余透射进入底层；透射光在底层发生标准PBR反射/散射后再次穿过顶层射出。
- 数学表达：f_clearcoat = F_coat * f_specular_coat + (1 - F_coat)^2 * f_base，其中F_coat是顶层的菲涅尔反射率，(1-F_coat)^2考虑了光进出顶层的两次透射。

### 次表面散射（SSS）模型

- 次表面散射描述光在半透明材料内部传输一定距离后从不同位置射出的现象。
- 典型应用：人类皮肤、蜡烛、玉石、大理石、牛奶、树叶。
- 标准PBR的Lambert Diffuse假设光在入射点附近散射（局部散射），无法模拟SSS的深度透光效果。
- 近似模型：
-   - Burley近似（Disney 2012）：用7个高斯核的加权叠加来模拟散射轮廓，计算效率高，效果接近参考解。
-   - Hanrahan-Krueger模型：基于偶极子（Dipole）近似的解析解，精度高但计算量大。
-   - 屏幕空间SSS（SSSSS）：在屏幕空间对Diffuse结果做模糊，受屏幕分辨率和深度不连续性影响。
- SSS的渲染方程扩展为BSSRDF：f(p_i, ω_i; p_o, ω_o)，引入了入射点p_i和出射点p_o的空间距离参数。

### 多层BRDF的混合策略

- 串行混合（Serial Layering）：光依次穿过每一层，每层按照菲涅尔比例分配反射和透射能量。
-   公式：f_total = f_top + T_top^2 * f_bottom / (1 - f_top * f_bottom)，其中T_top = 1 - F_top。
-   串行混合物理正确但计算复杂，分母中的f_top * f_bottom项（层间多次反射）通常被忽略。
- 并行混合（Parallel Mixing）：假设每层占据表面的一部分面积，按权重加权平均。
-   公式：f_total = w1 * f1 + w2 * f2 + ...，其中Σwi = 1。
-   并行混合简单但不物理正确，适用于无法明确分层的材质（如混合纤维的布料）。
- 加权混合（Weighted Blending）：UE5使用的实用方案，通过clearcoat参数控制顶层权重：f = clearcoat * f_coat + (1 - clearcoat) * f_base，并手动调整权重确保能量守恒。


## 🛠 工程实践

### 汽车漆面渲染（Clear Coat + Metallic Base）

- 汽车漆面是多层材质的经典案例：最外层是透明清漆（roughness≈0.1-0.3），中间是金属漆层（roughness≈0.3-0.6），最内层是底漆。
- 在UE5中，使用Clear Coat输入引脚即可启用：Clear Coat参数控制顶层强度（0-1），Clear Coat Roughness控制顶层粗糙度。
- 底层使用标准的Metallic-Roughness参数（Base Color、Metallic、Roughness）。
- Clear Coat的法线可以独立于底层法线（使用Clear Coat Normal Map），模拟清漆层的微观凹凸。
- 关键细节：Clear Coat层的F0固定为0.04（透明介质），roughness通常低于底层（清漆比底漆更光滑）。

### 皮肤渲染（SSS + Oil Layer）

- 人类皮肤的渲染需要至少两层：表皮层（SSS散射，产生血色和透光效果）和油脂层（薄薄的油膜，产生菲涅尔反射）。
- UE5的Subsurface Profile材质：通过Subsurface Color（散射颜色）和Opacity（散射距离）控制SSS效果。
- Unity HDRP的SSS：使用Subsurface Scattering Radius和Scattering Color参数。
- 皮肤渲染的关键参数：散射距离（红色约7mm、绿色约3mm、蓝色约1mm——红色光穿透最深，这就是手指透光时看起来红色的原因）。
- SSS的屏幕空间实现（SSSSS）：在Deferred Shading的Lighting Pass之后，对Diffuse结果按深度权重做7-tap高斯模糊。

### Split Sum在多层材质中的应用

- 对于Clear Coat材质，间接Specular需要考虑两层BRDF的叠加：
-   顶层IBL：使用Clear Coat的roughness采样预滤波贴图，F0=0.04。
-   底层IBL：使用Base的roughness采样预滤波贴图，F0由metallic决定。
-   总Specular IBL = F_coat * IBL_coat + (1-F_coat)^2 * IBL_base。
- 由于预滤波贴图是按roughness级别存储的，两层可以使用同一张预滤波贴图的不同LOD级别。
- BRDF LUT可以复用（与材质层数无关），但需要注意每层的NdotV可能不同（如果法线不同）。


## ⚠️ 踩坑经验

### 多层BRDF的能量守恒问题

- 简单的线性叠加 f_total = f1 + f2 会导致总反射率超过1（尤其在掠射角时，两层的菲涅尔反射率都接近100%）。
- 正确的串行混合需要考虑层间多次反射：f_total = f_top + T_top^2 * f_bottom * Σ(f_top * f_bottom)^n（无穷级数）。
- 工程中通常忽略高阶项（n≥2），使用 f_total ≈ f_top + T_top^2 * f_bottom 作为一阶近似。
- 即使是一阶近似，在掠射角时也可能出现能量略超1的情况（因为Schlick近似本身的误差），需要额外的能量钳制。
- 验证方法：在均匀白光下渲染roughness=0的多层球体，检查边缘是否出现过亮区域。

### SSSSS的屏幕空间瑕疵

- 屏幕空间SSS（SSSSS）在深度不连续处（物体边缘、遮挡边界）会产生「光泄漏」——模糊跨越深度边界，导致相邻物体被错误地照亮。
- 解决方案：在模糊时使用深度权重，深度差异大的像素不参与模糊。
- 另一个问题：SSSSS只能模拟屏幕上可见的散射，无法模拟物体背面的透光效果（如手指放在光源后面时的红色透光）。
- 对于需要精确SSS的场景（如角色特写），建议使用预积分的SSS纹理或光线追踪方案。
- 性能考虑：SSSSS的7-tap高斯模糊在4K分辨率下可能成为性能瓶颈，可以考虑降分辨率执行（半分辨率SSSSS）。

### Clear Coat法线与底层法线的冲突

- 如果Clear Coat层和底层使用不同的法线贴图（如清漆层有细微的橘皮纹理，底层有金属颗粒纹理），在交界处可能产生不自然的视觉冲突。
- 建议：Clear Coat法线扰动应小于底层法线扰动（清漆层通常比底漆更光滑）。
- 如果只有一张法线贴图，Clear Coat层可以使用平滑后的版本（通过降低法线贴图的强度或使用低频版本）。
- 在某些实现中，Clear Coat层不使用独立法线，而是与底层共享法线，简化实现并避免冲突。


## 🔮 延伸思考

### Disney Principled BRDF的Clear Coat和Sheen扩展

- Disney Principled BRDF（2012）定义了一套完整的PBR参数化模型，后续扩展包括：
- Clear Coat（2015扩展）：额外的透明层，参数包括clearcoat（0-1强度）和clearcoatGloss（光泽度）。
- Sheen（2015扩展）：用于模拟布料和绒毛材质的反射特性，使用Charlie NDF和独立的sheenColor参数。
- Thin Film（后续扩展）：薄膜干涉效果，通过thinFilmThickness参数控制。
- Transmission（后续扩展）：透明/半透明材质的透射，替代传统的opacity参数。
- 这些扩展被整合到Filament材质系统（Google开源）和MaterialX标准中，成为PBR材质建模的完整工具箱。

### 未来的材质建模方向

- 基于测量的材质表示：使用BRDF测量设备（如MERL gonioreflectometer）获取真实材质数据，直接用于渲染而非参数化模型。
- 神经网络材质模型：使用神经网络学习BRDF的表示和采样，如NeuralBRDF、Neural Material Representations。
- 参与介质（Participating Media）渲染：云、雾、烟雾、体积光等参与介质的实时渲染正在成为研究热点。
- 光谱渲染：用完整的光谱分布替代RGB三通道，可以模拟色散、荧光等波长相关现象。NVIDIA的RTX技术已经支持光谱追踪。
- 程序化材质生成：结合Procedural Noise和机器学习，自动生成物理正确的PBR材质贴图集（Albedo、Normal、Roughness等）。
- 这些方向共同推动着PBR从」基于物理的近似「向」基于物理的精确「演进。

---

[← 返回 光照与着色模型 题目列表](./index.md) | [返回总纲](../00-总纲.md)
