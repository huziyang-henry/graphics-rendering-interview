---
id: Q06.04
title: "Forward+（Forward Plus / Tiled Forward）渲染的原理是什么？它如何结合前向渲染和延迟渲染的优点？"
chapter: 6
chapter_name: "渲染架构与管线设计"
difficulty: advanced
knowledge_points:
  - id: "06.04"
    name: "Forward变体（Forward+/Clustered）"
tags: ["forward-plus", "tiled-forward", "light-culling", "tile"]
---

# Q06.04 Forward+（Forward Plus / Tiled Forward）渲染的原理是什么？它如何结合前向渲染和延迟渲染的优点？

**难度：** 🔵 高级
**所属章节：** [渲染架构与管线设计](./index.md)

---

## 🎯 结论

- Forward+ 通过 Tiled 或 Clustered Light Culling 预计算每个屏幕区域（Tile 或 Cluster）中影响的光源列表，在 Forward 渲染框架下将每个片元需要处理的光源数从全部光源降低到仅该区域的光源。
- 它结合了 Forward Rendering 的优势（MSAA 支持、半透明物体处理、低带宽开销）和 Deferred Rendering 的优势（多光源性能），是一种实用的折中方案。
- Forward+ 已被广泛采用，UE4/UE5 的 Forward Shading 模式默认使用 Clustered Forward，许多 3A 游戏在 PS5/Xbox Series X 上使用 Forward+ 作为主要渲染路径。

## 📐 原理解析

### Tiled Forward 渲染原理

- 将屏幕空间划分为固定大小的 Tile（通常 16×16 或 32×32 像素）。
- 使用 Compute Shader（或 Fragment Shader）遍历所有光源，判断每个光源影响哪些 Tile（基于光源的 Screen-Space Bounding Volume 与 Tile 的 AABB 相交测试）。
- 将每个 Tile 的光源索引列表写入一个全局的 Light Index List（存储在 SSBO 或 UAV 中）。
- 在 Forward 渲染的 Fragment Shader 中，根据当前片元所在的 Tile 查询 Light Index List，只遍历该 Tile 的光源计算光照。
- 复杂度从 O(Pixels × M) 降低到 O(Pixels × K)，其中 K 是每个 Tile 的平均光源数（通常 K << M）。

### Clustered Forward 渲染原理

- 在 Tiled 的基础上，将视锥体沿深度方向也进行切分，形成 3D Cluster（通常使用对数深度划分，近处 Cluster 薄、远处 Cluster 厚）。
- 光源与 Cluster 的相交测试使用球体-AABB 或锥体-AABB 测试，精确判断光源影响哪些 Cluster。
- Fragment Shader 中根据片元的屏幕位置和深度值确定所属 Cluster，查询该 Cluster 的光源列表。
- 解决了 Tiled Forward 在深度方向上的光源误归属问题（如远处的聚光灯覆盖整个 Tile 但只影响很小深度范围）。

### 与 Deferred 的对比

- Deferred：G-Buffer 写入开销大（带宽），但 Lighting Pass 中每个像素只需读取一次 G-Buffer，光照计算效率高。
- Forward+：无 G-Buffer 写入开销，但每个片元需要执行材质计算和光照计算。通过 Light Culling 减少光照遍历次数，在中等光源数量下性能接近 Deferred。
- Forward+ 的优势：支持 MSAA（无需 resolve 多个 RT）、支持半透明物体（同一渲染路径）、带宽开销低。


## 🛠 工程实践

### Tile 大小的选择

- 常见的 Tile 大小为 16×16 或 32×32 像素。较小的 Tile 提供更精细的光源剔除，但增加了 Tile 数量和 Light Index List 的存储开销。
- 32×32 是一个较好的平衡点：在 1080p 下约有 60×34 = 2040 个 Tile，Light Index List 的存储开销可控。
- Tile 大小的选择应考虑 Wave/Warp 的执行效率：确保一个 Wave 可以处理一个完整的 Tile 行（NVIDIA Warp 大小为 32，AMD Wavefront 大小为 64）。

### Light Index List 的存储格式

- 使用两个全局 Buffer：Light Grid（存储每个 Tile/Cluster 的光源数量和起始偏移）和 Light Index List（存储所有光源索引的扁平数组）。
- 在 Compute Shader 中使用 Atomic Counter 分配每个 Tile 的光源列表空间，或使用 Prefix Sum 预计算偏移。
- 每个 Tile 的最大光源数需要设置上限（如 256），超出时截断或使用降级策略（如 Fallback 到遍历所有光源）。

### 与 Forward 渲染的集成方式

- 在渲染管线的最前面插入一个 Light Culling Pass（Compute Shader），生成 Light Grid 和 Light Index List。
- Forward 渲染的 Fragment Shader 中，通过 SV_DispatchThreadID（Compute Shader）或 gl_FragCoord（Fragment Shader）计算当前片元所属的 Tile/Cluster。
- 从 Light Grid 中读取该 Tile/Cluster 的光源数量和起始偏移，从 Light Index List 中读取光源索引，遍历这些光源计算光照。
- 光源数据（位置、颜色、强度、类型等）存储在 Structured Buffer 或 SSBO 中，通过索引访问。


## ⚠️ 踩坑经验

### Tile 边界处的光源归属问题

- 当光源的 Screen-Space Bounding Volume 跨越多个 Tile 时，该光源会被添加到所有相交 Tile 的光源列表中，导致边缘片元的光照计算重复。
- 这不是正确性问题（光照结果仍然正确），但可能导致边缘 Tile 的光源数增加，影响性能。
- 优化方案：使用更精确的光源-Tile 相交测试（如将圆形光源近似为 Screen-Space 椭圆），减少误报。

### Cluster 的深度范围划分策略

- 均匀划分：将深度范围 [Near, Far] 等分为 N 份。简单但效率低，近处的 Cluster 过大（近处需要更高的精度），远处的 Cluster 过小。
- 对数划分：按对数间距划分深度，近处 Cluster 薄、远处 Cluster 厚，与人眼的深度感知匹配。
- 指数划分：结合对数和均匀划分的优点，在近处和远处都提供合理的 Cluster 粒度。
- 划分策略的选择需要根据具体场景的深度分布进行调整，没有通用的最优方案。

### Forward+ 的 Compute Shader 开销

- Light Culling 的 Compute Shader 开销与光源数量和 Tile/Cluster 数量成正比。在光源数量很大（> 1000）时，Culling 本身可能成为瓶颈。
- 优化手段：使用 Frustum Culling 预先剔除不在视锥体内的光源，减少需要测试的光源数。
- 使用 Indirect Dispatch 根据上一帧的光源分布动态调整 Compute Shader 的 Dispatch 参数。


## 🔮 延伸思考

### Forward+ vs Clustered Forward 的对比

- Tiled Forward（2D）：实现简单，适合光源分布较均匀的场景。但在深度方向上精度不足，远处的聚光灯可能被分配到不必要的 Tile。
- Clustered Forward（3D）：在深度方向上也进行切分，光源归属更精确。但 Cluster 数量增加（通常 16×16×32 = 8192 个 Cluster vs 2040 个 Tile），内存和计算开销更大。
- 在实际工程中，Clustered Forward 是更主流的选择，因为现代 GPU 的 Compute Shader 能力足以处理 3D Cluster 的 Culling 开销。

### Forward+ 在移动端的适用性

- 移动端 GPU 的 Compute Shader 能力有限（尤其是 OpenGL ES 3.1 的 Compute Shader 支持不完善），Forward+ 的 Compute Shader Culling 可能不是最优选择。
- 替代方案：使用 CPU 端的 Tiled Light Culling（将光源数据从 GPU 读回 CPU 进行 Culling，再将结果上传到 GPU），但 CPU-GPU 数据传输的延迟是瓶颈。
- WebGPU 环境下，Compute Shader 的支持更加标准化，Forward+ 在移动端浏览器中可能成为可行方案。

---

[← 返回 渲染架构与管线设计 题目列表](./index.md) | [返回总纲](../00-总纲.md)
