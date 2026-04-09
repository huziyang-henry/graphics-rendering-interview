---
id: Q01.02
title: "顶点着色器和片元着色器分别在管线的哪个阶段执行？它们的输入和输出分别是什么？"
chapter: 1
chapter_name: "渲染管线基础"
difficulty: beginner
knowledge_points:
  - id: "01.02"
    name: "顶点着色器与片元着色器"
tags: ["shader", "vertex-shader", "fragment-shader"]
---

# Q01.02 顶点着色器和片元着色器分别在管线的哪个阶段执行？它们的输入和输出分别是什么？

**难度：** 🟢 初级
**所属章节：** [渲染管线基础](./index.md)

---

## 🎯 结论

- 顶点着色器（VS）在几何阶段的最前端执行，是管线的第一个可编程阶段。
- 片元着色器（FS）在光栅化阶段执行，位于三角形遍历和属性插值之后。
- VS处理的是顶点级别的数据，FS处理的是像素级别的数据，两者通过Varying/Interpolant传递信息。

## 📐 原理解析

### 顶点着色器（Vertex Shader）

- 执行位置：几何阶段，在图元装配之前
- 输入：顶点属性（Position、Normal、UV、Tangent、Bone Weight等），Uniform/UBO（MVP矩阵、灯光参数等），SSBO（可选的大规模数据）
- 输出：必选——Clip Space位置（gl_Position / SV_Position）；可选——传递给FS的Varying（颜色、UV、法线、世界坐标等）
- 执行频率：每个顶点执行一次

### 片元着色器（Fragment/Pixel Shader）

- 执行位置：光栅化阶段，在三角形遍历和属性插值之后
- 输入：插值后的Varying（来自VS输出的透视校正插值结果），Uniform/UBO（材质参数、灯光参数等），采样器（Texture Sampler）
- 输出：必选——片元颜色（gl_FragColor / SV_Target[0]）；可选——深度值（gl_FragDepth / SV_Depth）
- 执行频率：每个覆盖的片元执行一次（受MSAA影响，每个子样本可能执行一次）

### Varying传递机制

- VS输出的Varying在光栅化阶段经过透视校正插值（Perspective-Correct Interpolation）后传递给FS
- 插值限定符：smooth（默认，透视校正）、flat（无插值）、noperspective（线性插值）
- flat限定符常用于整数属性（如材质ID），避免插值导致的无意义中间值


## 🛠 工程实践

- VS中做骨骼动画蒙皮时，需要注意Bone Weight的归一化（确保权重之和为1），否则变换后的顶点位置会偏移。
- 法线变换必须使用模型矩阵的逆转置矩阵（Normal Matrix = transpose(inverse(ModelMatrix))），否则非均匀缩放时法线方向会错误。
- FS中处理世界空间法线时，需要确认VS输出的是哪个空间的法线，避免在错误空间做光照计算。
- 在GLSL中，gl_FragCoord.xy是窗口坐标（左下角为原点），而SV_Position在HLSL中是屏幕坐标（左上角为原点），跨平台时需注意差异。

## ⚠️ 踩坑经验

- Varying插值的透视校正：在VS中传递1/w并在FS中除以插值后的1/w，才能获得正确的属性插值。现代GPU自动处理透视校正，但手动实现时容易遗漏。
- gl_FragCoord的精度问题：在移动端（尤其是高分辨率屏幕），gl_FragCoord.xy可能超出mediump float的精度范围，需要使用highp。
- VS输出过多Varying会增加寄存器压力，降低片元着色器的并行度。建议只传递FS真正需要的属性。
- FS中写入gl_FragDepth会导致Early-Z失效（因为深度值在FS执行前无法确定），应尽量避免。

## 🔮 延伸思考

- Geometry Shader（GS）和Tessellation Shader（TS）位于VS和FS之间，但它们在实际项目中使用频率很低。GS因为输出放大导致的性能问题被业界普遍避免；TS主要用于地形和角色细节增强，但Mesh Shader正在取代它。
- Mesh Shader将VS和部分GS/TS的功能合并，允许在GPU端直接生成、变换和剔除网格，是现代管线的重要演进方向。
- 为什么FS的执行频率远高于VS？因为一个三角形可能覆盖数十到数百个像素，FS的执行量通常是VS的10-100倍，所以FS的优化往往比VS更重要。

---

[← 返回 渲染管线基础 题目列表](./index.md) | [返回总纲](../00-总纲.md)
