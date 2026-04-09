---
id: Q01.07
title: "顶点着色器和片元着色器之间有哪些可选的可编程/固定功能阶段？它们各自的作用是什么？"
chapter: 1
chapter_name: "渲染管线基础"
difficulty: intermediate
knowledge_points:
  - id: "01.06"
    name: "可选管线阶段（曲面细分/几何着色器）"
tags: ["tessellation", "geometry-shader", "hull-shader"]
---

# Q01.07 顶点着色器和片元着色器之间有哪些可选的可编程/固定功能阶段？它们各自的作用是什么？

**难度：** 🟡 中级
**所属章节：** [渲染管线基础](./index.md)

---

## 🎯 结论

- VS和FS之间有两个可选的可编程阶段：曲面细分着色器（Tessellation Shader）和几何着色器（Geometry Shader）。
- 固定功能阶段包括图元装配（Primitive Assembly）和裁剪（Clipping），这些不可编程但可配置。
- 实际项目中，Geometry Shader因为性能问题极少使用，Tessellation Shader主要用于地形LOD，Mesh Shader正在取代两者。

## 📐 原理解析

### 曲面细分着色器（Tessellation Shader）

- 由三个阶段组成：Tessellation Control Shader（TCS/Hull Shader）→ Tessellation Primitive Generator（固定功能）→ Tessellation Evaluation Shader（TES/Domain Shader）
- TCS：指定细分因子（Inner/Outer Tessellation Levels），可逐顶点修改属性
- Primitive Generator：根据细分因子生成新的顶点（图元类型：三角形、四边形、等值线）
- TES：对新顶点执行位移计算（Displacement Mapping），确定最终位置
- 典型应用：地形LOD（根据距离动态调整网格密度）、角色细节增强、布料模拟

### 几何着色器（Geometry Shader）

- 输入：完整的图元（点/线/三角形及其邻接顶点）
- 输出：可以生成新的图元（点/线/三角形条带），也可以丢弃图元
- 输出放大（Output Amplification）：一个输入图元可以生成多个输出图元，这是GS性能瓶颈的根源
- 典型应用：法线可视化、粒子生成（从点生成四边形Billboard）、阴影体积（Shadow Volume）生成
- 实际项目中很少使用：因为输出放大导致串行化执行，无法充分利用GPU的并行计算能力

### 固定功能阶段

- 图元装配（Primitive Assembly）：将顶点按拓扑类型组装成图元
- 裁剪（Clipping）：在裁剪空间中剔除视锥体外图元
- 屏幕映射（Screen Mapping）：NDC到屏幕坐标的转换


## 🛠 工程实践

- Tessellation实现地形LOD：在TCS中根据相机距离计算细分因子，近处高细分远处低细分。TES中采样高度图进行顶点位移。需要注意相邻Patch的细分因子差异不能超过1，否则会产生裂缝（Crack）。
- Geometry Shader实现粒子效果：从VS输出的点（GL_POINTS）在GS中展开为面向相机的四边形（Billboard），附加粒子大小和旋转信息。但这种方式性能不如Instanced Quad + VS展开。
- 在Unity中，Tessellation通过Surface Shader的tessellate:函数名指令启用。在UE中，通过Domain Shader和Hull Shader直接编写。
- 现代引擎中，GS的典型替代方案：Instancing（实例化渲染）替代GS生成图元、Compute Shader替代GS的通用计算、Indirect Draw替代GS的输出放大。

## ⚠️ 踩坑经验

- Geometry Shader的性能陷阱：GS的输出必须写入缓冲区再被后续阶段消费，无法流水线化。当输出放大比高时（如1个三角形生成20个三角形），GS成为严重的性能瓶颈。经验法则：GS的输出总量应尽量接近输入总量。
- Tessellation的Crack问题：相邻Patch的细分因子不同时，共享边上的顶点数量不匹配导致裂缝。解决方案包括：限制相邻因子差值<=1、使用PN-AEN（PN Triangles with Adaptive Edge Normals）等算法。
- Tessellation的过度细分：在近处使用过高的细分因子会导致三角形数量爆炸，VS和TES的执行次数剧增。需要设置合理的最大细分因子。
- GS在AMD GPU上的性能尤其差：AMD的GCN/RDNA架构对GS的支持不如NVIDIA，在某些情况下GS的性能可能比替代方案慢10倍以上。

## 🔮 延伸思考

- 为什么业界普遍避免使用Geometry Shader？核心原因是GS的输出放大破坏了GPU的SIMT并行模型。GPU的并行效率依赖于大量独立的线程执行相同代码，而GS的串行输出放大机制与此相矛盾。
- Mesh Shader（NVIDIA Turing+ / DX12 / Vulkan）是GS和TS的现代替代品：它将VS、GS和TS的功能合并为一个可编程阶段，支持Task Shader（工作组级别）+ Mesh Shader（网格级别）的两级结构。Mesh Shader可以在GPU端直接生成、变换和剔除网格，且没有GS的串行化瓶颈。
- 在UE5的Nanite虚拟几何体系统中，Mesh Shader被用于软件光栅化：将Cluster级别的网格数据通过Mesh Shader处理，在Compute Shader中进行光栅化，实现了传统管线无法达到的三角形吞吐量。

---

[← 返回 渲染管线基础 题目列表](./index.md) | [返回总纲](../00-总纲.md)
