---
id: Q06.06
title: "Clustered Rendering相比Tiled Rendering有什么改进？它如何处理光源在不同深度范围的归属问题？"
chapter: 6
chapter_name: "渲染架构与管线设计"
difficulty: advanced
knowledge_points:
  - id: "06.04"
    name: "Forward变体（Forward+/Clustered）"
tags: ["clustered", "tiled", "light-assignment", "depth"]
---

# Q06.06 Clustered Rendering相比Tiled Rendering有什么改进？它如何处理光源在不同深度范围的归属问题？

**难度：** 🔵 高级
**所属章节：** [渲染架构与管线设计](./index.md)

---

## 🎯 结论

- Clustered Rendering 将视锥体沿深度方向也进行切分，形成 3D Cluster（三维体素），解决了 Tiled Rendering 在深度方向上的光源误归属问题。
- Tiled Rendering 只做 2D 屏幕空间划分，远处的聚光灯可能覆盖整个 Tile 但只影响很小深度范围，导致该 Tile 的所有片元都遍历这个光源（实际上大部分片元不受影响）。
- Clustered Rendering 通过精确的光源-Cluster 相交测试，确保每个 Cluster 只包含真正影响它的光源，进一步减少每个片元的光照计算量。

## 📐 原理解析

### Tiled Rendering 的深度方向缺陷

- Tiled Rendering 将屏幕划分为 2D Tile，光源与 Tile 的相交测试基于光源的 Screen-Space Bounding Volume（投影后的 2D 形状）。
- 对于点光源和聚光灯，其影响范围是一个 3D 球体或锥体，投影到 2D 后可能覆盖很大的面积，但实际只影响很小的深度范围。
- 例如：一个远处的聚光灯，其投影覆盖了整个 Tile，但该 Tile 中大部分片元的深度不在聚光灯的范围内。Tiled Rendering 会将该光源添加到 Tile 的光源列表中，导致不必要的遍历。
- 在光源分布不均匀的场景中（如远处的射灯、深处的点光源），Tiled Rendering 的光源剔除效率显著下降。

### Clustered Rendering 的 3D 划分

- 将视锥体沿 X、Y、Z 三个方向切分为 3D Cluster 网格。X 和 Y 方向对应屏幕空间的 Tile 划分，Z 方向对应深度切分。
- 深度切分通常使用对数划分（Logarithmic Split）：近处的 Cluster 薄（需要更高的深度精度），远处的 Cluster 厚（深度精度要求低）。公式：z_split = near × (far/near)^(i/N)，其中 i 为 Cluster 索引，N 为深度方向的总 Cluster 数。
- 每个 Cluster 是一个视锥体裁剪后的 Frustum（梯形体），可以用 AABB 近似以简化相交测试。
- 光源与 Cluster 的相交测试：点光源使用球体-AABB 测试，聚光灯使用锥体-AABB 测试，方向光影响所有 Cluster（或使用 Cascade Shadow Map 的划分方式）。

### 光源-Cluster 相交测试的数学基础

对于点光源（球体）与 Cluster（AABB）的相交测试，计算球心到 AABB 的最近点距离，如果距离小于球体半径则相交。对于聚光灯（锥体）与 Cluster（AABB）的相交测试，可以使用分离轴定理（SAT）或保守的锥体-AABB 包围测试。这些测试在 Compute Shader 中高效执行，利用 GPU 的大规模并行能力。


## 🛠 工程实践

### Cluster 的 3D 网格划分策略

- 典型的 Cluster 网格：16×16×32 = 8192 个 Cluster（X/Y 方向 16 个 Tile，Z 方向 32 层）。
- 深度方向的对数划分参数需要根据场景的 Near/Far 平面和光源分布进行调整。Near/Far 比值越大（如 0.1m 到 1000m），深度方向的 Cluster 数量需要越多。
- 一些实现使用自适应深度划分：根据上一帧的光源分布动态调整 Cluster 的深度边界，提高光源剔除的效率。
- Cluster 的数据结构通常使用 Flat Array 存储，每个 Cluster 的光源列表通过 Offset 和 Count 索引到全局的 Light Index List。

### 光源-Cluster 相交测试的优化

- 球体-AABB 测试：计算球心到 AABB 最近点的距离，比较与球体半径。这是最常用的测试，适用于点光源。
- 锥体-AABB 测试：将锥体近似为一个截头锥体（Frustum），使用分离轴定理（SAT）进行精确测试。计算开销较大，但可以避免误报。
- 保守测试：将光源的 Bounding Volume 扩大一定比例（如 10%），减少漏报（False Negative）但增加误报（False Positive）。在性能敏感的场景中，保守测试是更好的选择。
- 使用 SIMD 指令（如 SSE/NEON）并行测试多个光源与一个 Cluster 的相交性，提高 Compute Shader 的吞吐量。

### Cluster 数据结构的内存布局

- Light Grid：一个 3D 纹理或 Structured Buffer，存储每个 Cluster 的光源数量和起始偏移。大小为 (TileX × TileY × ClusterZ) × sizeof(uint2)。
- Light Index List：一个全局的扁平数组，存储所有 Cluster 的光源索引。使用 Atomic Append 或 Prefix Sum 分配空间。
- 在 Vulkan 中，Light Grid 可以存储在 Storage Image 中（支持 3D 纹理的硬件加速采样），Light Index List 存储在 Storage Buffer 中。
- 内存开销估算：假设 16×16×32 = 8192 个 Cluster，每个 Cluster 平均 8 个光源，Light Index List 大小约 8192 × 8 × 4 = 256KB，Light Grid 大小约 8192 × 8 = 64KB，总计约 320KB，完全在可接受范围内。


## ⚠️ 踩坑经验

### Cluster 数量增加导致的内存和计算开销

- 相比 Tiled Rendering 的 ~2000 个 Tile，Clustered Rendering 的 ~8000 个 Cluster 增加了 4 倍的 Light Grid 存储和光源剔除计算量。
- 在光源数量很大（> 1000）时，Compute Shader 的 Culling Pass 开销可能成为瓶颈。需要优化相交测试的实现（如使用更快的近似测试、减少不必要的精度）。
- Light Index List 的总大小可能超过 GPU 的 Shared Memory 限制，需要使用 Global Memory，访问延迟较高。

### 深度划分的粒度选择

- 深度方向的 Cluster 数量太少（如 8 层）：光源剔除精度不足，接近 Tiled Rendering 的效果。
- 深度方向的 Cluster 数量太多（如 64 层）：Light Grid 和 Culling 计算的开销增加，但收益递减（大部分 Cluster 的光源数已经很少）。
- 经验值：16-32 层是一个较好的平衡点，具体取决于场景的深度范围和光源分布。
- 对数划分的底数（Near/Far 比值的指数）需要根据场景调整，固定值可能不适用于所有场景。

### Clustered Rendering 的 Light Volume 优化

- 对于体积光（Volumetric Light），Clustered Rendering 可以复用 Cluster 的 3D 划分，在每个 Cluster 中计算体积光的散射和衰减。
- 但体积光的计算开销与 Cluster 数量成正比，需要使用 Ray Marching 等近似算法减少计算量。
- 体积光的 Cluster 划分可能与光源 Culling 的 Cluster 划分不一致，需要额外的插值或采样。


## 🔮 延伸思考

### Clustered Rendering 与 Forward+ 的关系

Clustered Rendering 是 Forward+ 的一种具体实现方式。Forward+ 是一个更广泛的概念，指在 Forward 渲染框架下使用预计算的光源剔除策略。Tiled Forward（2D）和 Clustered Forward（3D）都是 Forward+ 的子集。在实际工程中，"Forward+" 通常指 Tiled Forward，而 "Clustered Forward" 或 "Clustered Rendering" 指的是 3D 版本。UE4/UE5 使用的是 Clustered Forward，而非简单的 Tiled Forward。

### WebGL 环境下的 Clustered 实现

- WebGL 2.0 支持 Compute Shader（通过 EXT_disjoint_timer_query_webgl2 等扩展），但支持不完善，很多移动浏览器不支持。
- 替代方案：使用 Transform Feedback 或 Multiple Render Targets 在 Fragment Shader 中模拟 Compute Shader 的功能。
- WebGPU 原生支持 Compute Shader，Clustered Rendering 在 WebGPU 环境下的实现更加自然和高效。
- Web 环境下的带宽限制更加严格，Clustered Rendering 的 Light Index List 需要尽可能紧凑，使用 16-bit 索引替代 32-bit 索引。

---

[← 返回 渲染架构与管线设计 题目列表](./index.md) | [返回总纲](../00-总纲.md)
