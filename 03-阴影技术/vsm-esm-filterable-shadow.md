---
id: Q03.05
title: "VSM（Variance Shadow Map）和ESM（Exponential Shadow Map）的原理分别是什么？它们各自有什么优缺点？"
chapter: 3
chapter_name: "阴影技术"
difficulty: advanced
knowledge_points:
  - id: "03.02"
    name: "软阴影技术（PCF/PCSS/VSM/ESM）"
tags: ["vsm", "esm", "evsm", "filterable-shadow", "light-bleeding"]
---

# Q03.05 VSM（Variance Shadow Map）和ESM（Exponential Shadow Map）的原理分别是什么？它们各自有什么优缺点？

**难度：** 🔵 高级
**所属章节：** [阴影技术](./index.md)

---

## 🎯 结论

- VSM（Variance Shadow Map）和ESM（Exponential Shadow Map）是两种重要的可滤波阴影技术，它们的核心价值在于允许对Shadow Map进行硬件加速的预滤波（如Mipmap、双线性/三线性滤波），从而以较低的采样次数实现大范围的软阴影效果。
- VSM通过存储深度的均值和方差，利用Chebyshev不等式估算阴影概率；ESM通过指数函数将深度比较转化为乘法运算，使得深度缓冲可以被直接滤波。两者各有优劣，EVSM结合了两者的优势。

## 📐 原理解析

### VSM的数学原理

- VSM由Donnelly和Lauritzen于2006年提出。它不存储原始深度值，而是存储深度的前两阶矩：M1 = E(z)（深度均值）和M2 = E(z^2)（深度平方的均值）。方差可以通过Var(z) = M2 - M1^2计算。
- 在阴影查询时，VSM利用Chebyshev不等式来估算深度值小于给定阈值的概率上限：P(x <= t) <= pmax = Var(z) / (Var(z) + (t - M1)^2)，其中t是接收面的深度值。pmax即为阴影因子，值越小表示越可能处于阴影中。
- Chebyshev不等式给出的是概率上界，因此VSM倾向于低估阴影（即倾向于判定为光照），这就是光晕（Light Bleeding）问题的数学根源。

### ESM的数学原理

- ESM由Salvi等人于2008年提出。它将深度值z存储为指数形式：occluder = exp(-c * z)，其中c是一个正的常数参数。在阴影查询时，接收面的深度也被转换为指数形式：receiver = exp(-c * z_receiver)。
- 阴影判定变为：如果receiver < occluder（即exp(-c * z_receiver) < exp(-c * z_shadow)），则处于阴影中。由于指数函数的单调性，这等价于z_receiver > z_shadow，与标准Shadow Map的比较一致。
- 关键优势在于：当对存储的指数值进行线性滤波（如双线性插值或Mipmap）时，滤波后的结果近似等于指数深度的加权平均，而比较操作receiver * filtered_occluder < 1保持了物理上的合理性。这使得ESM的Shadow Map可以被预滤波。


## 🛠 工程实践

### VSM的Mipmap预滤波

VSM最大的工程优势在于支持Mipmap。通过为VSM的Shadow Map生成Mipmap链，可以在查询时使用不同层级的Mipmap来获得不同范围的滤波效果。远处的软阴影使用较粗的Mipmap层级（大范围模糊），近处的硬阴影使用较细的层级（小范围或无模糊）。这种方案只需一次Mipmap查找即可完成大范围滤波，远比PCF的多次采样高效。需要注意的是，VSM的Mipmap生成需要对M1和M2分别进行正确的均值合并，而非简单的双线性插值。

### ESM的c参数调优

- ESM的c参数控制指数函数的“陡峭程度”。c值越大，深度比较的边界越锐利，越接近硬阴影；c值越小，边界越柔和，但可能过度模糊阴影细节。
- c值的选择与场景的深度范围密切相关。对于深度范围较大的场景（如远距离方向光），需要较小的c值；对于深度范围较小的场景（如近距离点光源），可以使用较大的c值。
- 实践中通常需要根据具体场景进行调参，或使用自适应c值——根据当前级联的深度范围动态计算c值。

### EVSM——结合两者优势

Exponential Variance Shadow Map（EVSM）将VSM和ESM的思想结合。它存储深度的正指数和负指数的矩（即exp(c*z)和exp(-c*z)的均值和方差），从而同时具备VSM的概率估计能力和ESM的滤波友好性。EVSM的光晕问题比纯VSM轻得多，同时支持大范围的软阴影。UE4中的CSM就使用了EVSM作为可选的阴影滤波方案。


## ⚠️ 踩坑经验

### VSM的光晕问题在高对比度场景中的表现

VSM的光晕问题在遮挡物与接收面深度差较大时最为严重。例如，一个悬浮在高空的平台，其阴影投射到地面时，由于深度差很大，Chebyshev不等式的上界接近1，导致本应完全阴影的区域出现明显的漏光。常见的缓解方法包括：使用更高的矩（如4阶矩的Moment Shadow Maps）、对阴影因子进行幂次矫正（Shadow Factor = pmax^k，k通常取8-32）、或结合深度偏移来减小有效深度差。

### ESM的深度范围和c值的关系

- ESM的一个潜在问题是指数溢出。当深度值很大且c值也较大时，exp(-c*z)可能下溢为0，导致精度完全丧失。使用32位浮点格式（float32）可以缓解这个问题，但内存开销翻倍。
- 深度范围的非线性分布（如Reverse-Z）会影响ESM的精度分布。在Reverse-Z下，近处的深度值接近1，远处的深度值接近0，指数函数的行为与标准深度分布不同，需要相应调整c值。
- ESM在级联边界处可能出现不一致的阴影柔和度，因为不同级联的深度范围不同，对应的c值也不同。需要在级联过渡区域做平滑处理。


## 🔮 延伸思考

### Moment Shadow Maps（MSM）

Moment Shadow Maps由Peters等人于2015年提出，通过存储深度的更高阶矩（通常为4阶）来构建更精确的深度分布表示。4阶矩可以重构一个满足某些约束的深度分布函数，从而比Chebyshev不等式更准确地估算阴影概率。MSM在光晕问题上比VSM有显著改善，同时保持了可滤波的优势。其代价是存储量增加（4个通道存储4阶矩）和计算复杂度略高。MSM被认为是目前最好的可滤波阴影技术之一，在多个商业引擎中被采用。

---

[← 返回 阴影技术 题目列表](./index.md) | [返回总纲](../00-总纲.md)
