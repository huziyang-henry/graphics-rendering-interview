---
id: Q02.05
title: "微表面理论中的法线分布函数（NDF）有哪几种常见实现？GGX和Beckmann的区别是什么？"
chapter: 2
chapter_name: "光照与着色模型"
difficulty: intermediate
knowledge_points:
  - id: "02.05"
    name: "微表面理论（NDF/Fresnel/Geometry）"
tags: ["ndf", "ggx", "beckmann", "microfacet"]
---

# Q02.05 微表面理论中的法线分布函数（NDF）有哪几种常见实现？GGX和Beckmann的区别是什么？

**难度：** 🟡 中级
**所属章节：** [光照与着色模型](./index.md)

---

## 🎯 结论

- 法线分布函数（NDF）是微表面BRDF的核心组件，描述了微表面法线方向的统计分布。
- 常见的NDF实现包括：Beckmann分布、GGX（Trowbridge-Reitz）分布、Phong分布，以及Disney提出的各向异性NDF。
- GGX和Beckmann的核心区别在于高光「尾巴」的衰减速度：GGX的衰减更慢（长尾分布），高光边缘更柔和、更自然。
- GGX已成为业界标准NDF，被UE4/UE5、Unity HDRP、Filament等主流渲染引擎采用。

## 📐 原理解析

### NDF的数学定义与物理含义

- NDF $D(\mathbf{m})$ 定义为微表面法线 $\mathbf{m}$ 在方向空间中的概率密度函数（PDF），满足积分约束：

$$\int_{\Omega} D(\mathbf{m}) \, (\mathbf{n} \cdot \mathbf{m}) \, d\mathbf{m} = 1$$

- 注意积分中的 $(\mathbf{n} \cdot \mathbf{m})$ 因子——这是投影面积校正，确保在倾斜方向上的密度正确。
- NDF直接决定了高光的形状：$D$ 值大的方向上高光更亮，$D$ 值小的方向上高光更暗。
- $\text{roughness}$ 参数控制NDF的「集中度」：$\text{roughness}$ 越小，$D$ 函数越集中于镜面反射方向（高光越锐利）；$\text{roughness}$ 越大，$D$ 函数越分散（高光越柔和）。

### Beckmann分布

- Beckmann分布是最早用于计算机图形学的NDF之一，源自光学领域的粗糙表面散射理论。
- 公式：

$$D_{\text{Beckmann}}(\mathbf{m}) = \frac{\exp\left(-\tan^2(\theta_m) / \alpha^2\right)}{\pi \, \alpha^2 \, \cos^4(\theta_m)}$$

其中 $\theta_m$ 是 $\mathbf{m}$ 与法线 $\mathbf{n}$ 的夹角，$\alpha$ 是粗糙度参数。
- Beckmann分布基于高斯假设，假设微表面斜率的分布服从正态分布。
- 特点：高光衰减速度快，高光边缘锐利，在 $\text{roughness}$ 较大时高光消失得很快。
- 缺点：与真实材质测量数据（如MERL数据库）的匹配度不如GGX，高光看起来「过于干净」，缺乏真实材质的柔和过渡。

### GGX（Trowbridge-Reitz）分布

- GGX分布（也称为Trowbridge-Reitz分布）是当前最广泛使用的NDF。
- 公式：

$$D_{\text{GGX}}(\mathbf{m}) = \frac{\alpha^2}{\pi \left((\mathbf{n} \cdot \mathbf{m})^2 (\alpha^2 - 1) + 1\right)^2}$$

其中 $\alpha = \text{roughness}^2$。
- GGX的核心特征是「长尾」（long tail）：在远离镜面反射方向的角度上，$D$ 值衰减得比Beckmann慢得多。
- 这意味着GGX的高光边缘更柔和、过渡更自然，更接近真实世界中观察到的高光形态。
- 数学上，GGX在 $\theta_m \to 90°$ 时的衰减速度约为 $\cos^{-4}(\theta_m)$，而Beckmann约为 $\exp(-\tan^2(\theta_m)/\alpha^2)$，前者衰减更慢。

### 各向异性NDF

- 各向异性NDF在两个正交方向上有不同的粗糙度参数（$\alpha_x$ 和 $\alpha_y$），可以模拟拉丝金属、头发、丝绸等方向性材质。
- GGX的各向异性扩展：将 $\alpha$ 替换为基于切线方向的椭圆分布：

$$\alpha(\varphi) = \sqrt{\cos^2\varphi \cdot \alpha_x^2 + \sin^2\varphi \cdot \alpha_y^2}$$

- Disney Principled BRDF的各向异性参数通过 $\text{anisotropic}$（0-1）和tangent方向控制，内部映射为 $\alpha_x = \alpha / (1 - \text{anisotropic} \times 0.9)$ 和 $\alpha_y = \alpha \times (1 - \text{anisotropic} \times 0.9)$。
- 各向异性NDF在头发渲染（Kajiya-Kay模型）和布料渲染中尤为重要。


## 🛠 工程实践

### GGX NDF的GLSL实现

- 标准GGX实现：float D_GGX(float NdotH, float roughness) { float a = roughness * roughness; float a2 = a * a; float d = (NdotH * NdotH) * (a2 - 1.0) + 1.0; return a2 / (PI * d * d); }
- 注意 $\alpha = \text{roughness}^2$ 的映射——这是Disney/UE4的约定。如果直接使用 $\text{roughness}$ 作为 $\alpha$，高光会偏窄。
- 在移动端，可以用D_GGX的近似版本减少除法运算：利用rcp（倒数指令）替代除法。
- 各向异性GGX需要在切线空间中计算，需要额外的tangent和bitangent输入。

### NDF在预滤波环境贴图中的应用

- IBL的预滤波环境贴图通过卷积NDF来生成不同 $\text{roughness}$ 级别的模糊环境反射。
- 卷积过程：对环境贴图的每个像素，按照NDF $D(\mathbf{h})$ 的权重分布对周围像素进行加权平均。
- 不同mipmap层级对应不同 $\text{roughness}$ 值：level 0 = $\text{roughness}\ 0$（原始贴图），最高level = $\text{roughness}\ 1$（最大模糊）。
- UE4使用Importance Sampling（重要性采样）来高效地执行这一卷积，通常需要2048-4096个采样才能获得低噪声的结果。


## ⚠️ 踩坑经验

### NDF的归一化

- NDF必须满足归一化约束：

$$\int_{\Omega} D(\mathbf{m}) \, (\mathbf{n} \cdot \mathbf{m}) \, d\mathbf{m} = 1$$

如果NDF未正确归一化，BRDF的能量将不守恒。
- Beckmann和GGX的公式已经包含了归一化常数（分母中的 $\pi \alpha^2$），直接使用即可。
- 但如果自定义NDF（如修改GGX的指数或添加自定义分布），必须重新推导归一化常数。
- 验证方法：在均匀环境光下渲染 $\text{roughness}=1$ 的球体，如果整体亮度明显偏亮或偏暗，说明NDF可能未正确归一化。

### GGX长尾导致的Firefly问题

- GGX的长尾特性在低 $\text{roughness}$ 时可能导致Firefly（极亮像素）：当 $\mathbf{n} \cdot \mathbf{h}$ 接近0时，$D_{\text{GGX}}$ 的值虽然小但不为零，乘以高强度的环境光像素后可能产生异常亮点。
- 在路径追踪中，GGX的长尾还导致方差增大，需要更多采样才能收敛。
- 解决方案：在NDF计算中添加阈值裁剪（如 $\mathbf{n} \cdot \mathbf{h} < 0.001$ 时返回0），或使用GGX的修改版本（如GGX with Smith height-correlated masking）。
- 在实时渲染中，firefly问题通常不严重，因为屏幕空间的光照积分已经被离散化为逐像素计算。但在预滤波环境贴图的生成过程中可能需要额外处理。

### $\text{roughness}=0$ 时的NDF退化

- 当 $\text{roughness}=0$ 时，GGX退化为Dirac delta函数 $D(\mathbf{m}) = \delta(\mathbf{m} - \mathbf{n})$，数学上表示所有微表面法线完全一致。
- 在离散计算中，这会导致数值问题：$D$ 值在 $\cos\theta_{N}=1$ 时为无穷大，在其他位置为0。
- 工程解决方案：将 $\text{roughness}$ 钳制为最小值（如0.04），或对 $\text{roughness}=0$ 的情况做特殊处理（直接返回环境贴图的镜面反射采样）。


## 🔮 延伸思考

### 为什么GGX成为业界标准？

- GGX被广泛采用的原因：与MERL BRDF数据库中100种真实材质的测量数据拟合度最好（优于Beckmann和Phong分布）。
- GGX的长尾特性更符合真实世界观察：真实材质的高光边缘不是锐利的截止，而是逐渐衰减的过渡。
- GGX有解析的Smith G项和PDF表达式，便于重要性采样和预计算，工程实现友好。
- Walter等人2007年的论文《Microfacet Models for Refraction through Rough Surfaces》
- 系统地比较了各种NDF，GGX在精度和效率的综合表现上最优。
- Naty Hoffman在SIGGRAPH渲染课程
- 课程中的对比实验也证实了GGX的视觉优势。

### Charlie Sheen和NDF的关系

- Charlie Sheen是Disney Principled BRDF中用于模拟布料（织物）高光的特殊NDF。
- 布料的高光与常规GGX不同：布料高光更宽、更均匀，没有明显的镜面反射峰值。
- Charlie Sheen NDF基于简化的Gaussian分布：

$$D_{\text{Charlie}} = \frac{1 + 2 \cdot \text{roughness}}{2\pi} \cdot (1 - \cos\theta_{N})^{2 \cdot \text{roughness}}$$
- 与GGX相比，Charlie的高光峰值更低但覆盖范围更广，更接近布料的纤维散射特性。
- 在UE5的Cloth Shader和Unity HDRP的Fabric材质中，都提供了Charlie Sheen作为布料高光的选项。

---

[← 返回 光照与着色模型 题目列表](./index.md) | [返回总纲](../00-总纲.md)
