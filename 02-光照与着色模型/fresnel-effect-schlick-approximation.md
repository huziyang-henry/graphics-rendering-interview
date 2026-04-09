---
id: Q02.06
title: "菲涅尔效应（Fresnel Effect）在PBR中如何建模？Schlick近似公式的优缺点是什么？"
chapter: 2
chapter_name: "光照与着色模型"
difficulty: intermediate
knowledge_points:
  - id: "02.05"
    name: "微表面理论（NDF/Fresnel/Geometry）"
tags: ["fresnel", "schlick", "f0", "pbr"]
---

# Q02.06 菲涅尔效应（Fresnel Effect）在PBR中如何建模？Schlick近似公式的优缺点是什么？

**难度：** 🟡 中级
**所属章节：** [光照与着色模型](./index.md)

---

## 🎯 结论

- 菲涅尔效应描述了表面反射率随入射角变化的物理现象：正射时反射率最低（$F_0$），掠射角时反射率趋近于100%（$F_{90}$）。
- PBR中使用Schlick近似公式来高效计算菲涅尔效应：$F(\mathbf{v}, \mathbf{h}) = F_0 + (1 - F_0)(1 - (\mathbf{v} \cdot \mathbf{h}))^5$。
- Schlick近似的优点是计算效率极高（仅需一次pow5运算）、精度在大多数场景下足够、易于集成到Shader中。
- 缺点是在掠射角附近精度下降、无法模拟偏振效应、对某些特殊材质（如金属的色散效应）不够准确。

## 📐 原理解析

### 菲涅尔效应的物理原理

- 菲涅尔效应由Augustin-Jean Fresnel在19世纪提出，描述了光在两种介质界面上的反射和透射比例随入射角变化的关系。
- 当光从空气（$n=1.0$）射入材质表面时，一部分光被反射，一部分被折射（进入材质内部）。
- 在正射（入射角=0度）时，反射率最低，记为 $F_0$（Fresnel reflectance at normal incidence）。
- 随着入射角增大，反射率单调递增，在掠射角（入射角→90°）时趋近于100%。
- $F_0$ 由两种介质的折射率决定：

$$F_0 = \left(\frac{n_1 - n_2}{n_1 + n_2}\right)^2$$

对于空气-材质界面：

$$F_0 = \left(\frac{1 - n}{1 + n}\right)^2$$

### 精确Fresnel方程

- 对于非偏振光，精确的Fresnel反射率为 $R_s$ 和 $R_p$ 的平均值：

$$F = \frac{R_s + R_p}{2}$$

- $R_s$（s偏振）：

$$R_s = \left(\frac{n_1 \cos\theta_i - n_2 \cos\theta_t}{n_1 \cos\theta_i + n_2 \cos\theta_t}\right)^2$$

- $R_p$（p偏振）：

$$R_p = \left(\frac{n_2 \cos\theta_i - n_1 \cos\theta_t}{n_2 \cos\theta_i + n_1 \cos\theta_t}\right)^2$$

- 其中 $\theta_t$ 由Snell定律确定：$n_1 \sin\theta_i = n_2 \sin\theta_t$。
- 对于金属，折射率为复数 $n = n + ik$（$k$ 为消光系数），Fresnel方程更复杂，且 $F_0$ 与波长相关（导致有色反射）。
- 精确Fresnel方程计算量大（涉及复数运算和三角函数），在实时渲染中通常使用近似。

### Schlick近似公式

- Christophe Schlick于1994年提出的近似公式：

$$F(\theta) = F_0 + (1 - F_0)(1 - \cos\theta)^5$$

- 在PBR的BRDF上下文中，$\theta$ 是视线方向 $\mathbf{v}$ 和半程向量 $\mathbf{h}$ 之间的夹角，因此公式写为：

$$F(\mathbf{v}, \mathbf{h}) = F_0 + (F_{90} - F_0)(1 - (\mathbf{v} \cdot \mathbf{h}))^5$$
- $F_{90}$ 通常设为1.0（掠射角全反射），但某些实现允许 $F_{90} < 1.0$ 来模拟特殊材质。
- Schlick近似在大多数入射角范围内与精确Fresnel方程的误差小于1%，仅在掠射角附近（>85°）误差增大到约5%。
- 计算优势：仅需一次减法、一次乘法和一次pow5运算。pow5可以通过两次平方 $(x \cdot x) \cdot (x \cdot x) \cdot x$ 来实现，避免昂贵的pow函数调用。


## 🛠 工程实践

### 金属和非金属的 $F_0$ 值确定

- 非金属（绝缘体）的 $F_0$ 值范围很窄，约0.02-0.08。在Metallic-Roughness工作流中，统一取 $F_0 = 0.04$（对应折射率约1.5，覆盖大多数常见非金属）。
- 金属（导体）的 $F_0$ 值范围较广，约0.5-1.0，且与波长相关。在Metallic-Roughness工作流中，$F_0$ 直接从Base Color贴图中采样（金属区域的Base Color存储的就是 $F_0$ 值）。
- 常见材质的 $F_0$ 参考值：水0.02、皮肤0.028、塑料0.03-0.05、玻璃0.04、钻石0.044、金(0.95,0.64,0.54)、铝(0.91,0.92,0.92)、铜(0.95,0.64,0.54)。
- 在Shader中的实现：`vec3 F0 = mix(vec3(0.04), albedo.rgb, metallic);` 然后应用Schlick近似。

### $F_{90}$ 的处理

- 标准PBR中 $F_{90} = 1.0$，但某些情况下需要调整：
- 对于非常粗糙的非金属表面，$F_{90}$ 可以小于1.0（因为粗糙表面的掠射角反射率可能达不到100%）。
- Karis（Epic Games）建议使用 $F_{90} = \text{clamp}(F_0 \times 50.0,\ 0.0,\ 1.0)$ 来避免非金属在掠射角时出现不自然的强反射。
- 在多层材质中（如Clear Coat），每层有自己的 $F_0$ 和 $F_{90}$，需要分层计算菲涅尔效应。

### GLSL/HLSL实现

vec3 fresnelSchlick(float cosTheta, vec3 F0, vec3 F1) { return F0 + (F1 - F0) * pow(clamp(1.0 - cosTheta, 0.0, 1.0), 5.0); }

### 优化版本（避免pow）

vec3 fresnelSchlickFast(float cosTheta, vec3 F0, vec3 F1) { float t = clamp(1.0 - cosTheta, 0.0, 1.0); float t2 = t * t; float t5 = t2 * t2 * t; return F0 + (F1 - F0) * t5; }


## ⚠️ 踩坑经验

### 忽略Fresnel导致金属材质异常

- 如果Shader中没有实现菲涅尔效应，金属材质在掠射角会显得暗淡（本应增强的反射缺失），而非金属在掠射角会显得过于透明。
- 这是传统Blinn-Phong材质看起来「塑料感」的根本原因之一——缺少菲涅尔效应使得所有角度的反射率相同。
- 在PBR管线中，菲涅尔效应不仅影响Specular强度，还影响Diffuse和Specular的能量分配（$k_d = 1 - F$），因此缺失Fresnel会导致Diffuse能量也出错。
- 调试方法：渲染一个金属球体，从正射旋转到掠射角，如果反射率没有明显增强，说明Fresnel未正确实现。

### 多层材质的Fresnel叠加

- 对于多层材质（如汽车漆面：透明清漆层 + 金属底色层），每层都有独立的菲涅尔效应。
- 简单叠加两层Fresnel会导致总反射率超过100%，违反能量守恒。
- 正确的做法是使用串行菲涅尔公式：光先经过顶层界面的菲涅尔反射/折射，折射部分再经过底层界面的菲涅尔反射/折射。
- UE5的Clear Coat实现使用了简化的两层模型：顶层使用独立的 $\text{roughness}$ 和 $F_0=0.04$，底层使用标准的PBR BRDF，两层通过能量加权混合。

### Schlick近似的精度局限

- Schlick近似在入射角>85°时误差增大，对于掠射角反射的精确模拟不够准确。
- 对于某些高折射率材质（如钻石 $n=2.42$，$F_0=0.17$），Schlick近似的误差比低折射率材质更明显。
- 如果需要更高的精度，可以使用Fresnel方程的精确实现（涉及sincos和sqrt），但计算量增加约3-5倍。
- 在离线渲染（如Arnold、RenderMan）中通常使用精确Fresnel，在实时渲染中Schlick近似是标准选择。


## 🔮 延伸思考

### 薄膜干涉（Thin-film Interference）的Fresnel建模

- 薄膜干涉发生在薄透明层覆盖在材质表面的情况（如肥皂泡、油膜、昆虫翅膀、某些宝石）。
- 物理原理：光在薄膜上下两个界面发生多次反射，不同波长的光因光程差不同而产生相长或相消干涉，呈现彩虹色。
- 建模方法：薄膜的 $F_0$ 变为波长的函数：

$$F_0(\lambda) = \frac{2 \, n_{\text{film}} \, n_{\text{base}} \, \cos(\delta / 2)}{n_{\text{film}}^2 + n_{\text{base}}^2} \times \cdots$$

其中 $\delta$ 是光程差相关的相位差。
- UE5从4.27版本开始支持Thin Film材质输入，通过薄膜厚度参数控制干涉色的变化。
- glTF的KHR_materials_iridescence扩展也定义了薄膜干涉的参数化接口。

### Iridescence效果实现

- Iridescence（虹彩效果）是薄膜干涉的视觉表现，在渲染中可以通过以下方式实现：
-   - 预计算薄膜干涉的 $F_0$ 光谱，将其映射到RGB空间。
-   - 使用薄膜厚度贴图（Thin Film Thickness Map）控制不同区域的干涉色。
-   - 在Shader中实时计算干涉色（计算量较大，适合桌面端）。
- Belcour和Barla在SIGGRAPH 2017的论文
- 中提出了高效的薄膜干涉近似算法。
- Unity HDRP从2021.2版本开始支持Iridescence，通过Material Iridescence参数和厚度贴图控制。

---

[← 返回 光照与着色模型 题目列表](./index.md) | [返回总纲](../00-总纲.md)
