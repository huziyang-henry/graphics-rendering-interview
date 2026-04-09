---
id: Q01.05
title: "光栅化阶段具体做了哪些工作？三角形设置（Triangle Setup）和三角形遍历（Triangle Traversal）分别是什么？"
chapter: 1
chapter_name: "渲染管线基础"
difficulty: intermediate
knowledge_points:
  - id: "01.04"
    name: "光栅化原理"
tags: ["rasterization", "triangle-setup", "triangle-traversal"]
---

# Q01.05 光栅化阶段具体做了哪些工作？三角形设置（Triangle Setup）和三角形遍历（Triangle Traversal）分别是什么？

**难度：** 🟡 中级
**所属章节：** [渲染管线基础](./index.md)

---

## 🎯 结论

- 光栅化是将矢量图元（三角形、线、点）转换为离散片元（Fragment）的过程，是连接几何处理和像素着色之间的桥梁。
- 三角形设置计算三角形边方程和属性梯度；三角形遍历判断每个像素是否被三角形覆盖并生成片元。
- 属性插值对顶点属性进行透视校正插值，确保纹理映射和颜色过渡的正确性。

## 📐 原理解析

### 三角形设置（Triangle Setup）

- 接收裁剪后的三角形三个顶点的屏幕坐标和属性值
- 计算边方程（Edge Function）：对于三角形的每条边，计算方程 E(x,y) = a*x + b*y + c
- 边方程的符号表示点在边的哪一侧：同侧为正，异侧为负
- 计算属性梯度（Attribute Gradients）：对于每个Varying属性，计算其在屏幕空间x和y方向的变化率（ddx、ddy）
- 这些梯度用于后续的属性插值：attribute(x,y) = attribute_0 + gradient_x * dx + gradient_y * dy

### 三角形遍历（Triangle Traversal）

- 遍历三角形的包围盒（Bounding Box）内的所有像素
- 对每个像素中心，使用边方程判断是否在三角形内部（三个边方程同号）
- 覆盖判断规则：左上角填充规则（Top-Left Rule）——共享边上的像素只被一个三角形拥有，避免重复覆盖
- 对覆盖的像素生成片元（Fragment），附带插值后的属性值
- 现代GPU通常以2x2像素块（Quad）为单位进行处理，以便计算ddx/ddy用于纹理mipmap选择

### 属性插值

- 透视校正插值（Perspective-Correct Interpolation）：在屏幕空间线性插值会导致近大远小的属性失真
- 正确方法：先对1/w和attribute/w进行线性插值，再在FS中除以插值后的1/w
- 重心坐标插值：attribute = a0*lambda0 + a1*lambda1 + a2*lambda2，其中lambda为重心坐标权重
- 现代GPU硬件自动处理透视校正插值，开发者通常无需手动计算


## 🛠 工程实践

- 光栅化规则对像素覆盖率的判断影响MSAA的效果：1x MSAA（无抗锯齿）只检查像素中心，2x/4x/8x MSAA在像素内多个采样点分别判断覆盖。
- Conservative Rasterization（保守光栅化）会将部分覆盖的像素也标记为覆盖，用于生成Voxel、碰撞检测等需要精确轮廓的场景。DX12和Vulkan都支持此扩展。
- 在Compute Shader中模拟光栅化时，需要手动实现边方程和覆盖率测试。常用于OIT（Order-Independent Transparency）和Voxelization等特殊渲染技术。
- RenderDoc的Raster视图可以直观查看每个三角形的覆盖像素和属性插值结果，是调试光栅化问题的利器。

## ⚠️ 踩坑经验

- 像素中心对齐问题：在OpenGL中，像素中心在(0.5
- 共享边上的像素归属：两个相邻三角形共享一条边时，Top-Left Rule决定哪些像素属于哪个三角形。如果渲染顺序不同，可能导致抗锯齿的覆盖信息不一致。
- 2x2 Quad执行模型：即使只有1个像素被三角形覆盖，GPU也会执行2x2=4个片元着色器调用（未覆盖的像素结果被丢弃）。这保证了ddx/ddy的正确性，但也意味着边缘像素有3倍的计算浪费。
- 超大三角形的性能问题：一个覆盖整个屏幕的三角形（如全屏Quad）会导致所有像素片元同时执行FS，可能造成片元处理器的瞬时负载过高。

## 🔮 延伸思考

- 光栅化顺序对性能的影响：虽然GPU不保证片元的执行顺序，但深度测试的顺序影响Early-Z的效率。从前到后（Front-to-Back）渲染可以最大化Early-Z的剔除效果。
- Conservative Rasterization在Voxel Cone Tracing和软件光栅化（如Nanite的软件光栅化器）中有重要应用。它确保了薄壁物体不会在Voxel化过程中丢失。
- 未来的渲染管线中，光栅化可能逐渐被混合光栅化-光线追踪管线取代。NVIDIA的RTX DI（Direct Illumination）和UE5的Lumen都展示了这种混合架构的潜力。

---

[← 返回 渲染管线基础 题目列表](./index.md) | [返回总纲](../00-总纲.md)
