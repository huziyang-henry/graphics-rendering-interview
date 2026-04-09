---
id: Q08.01
title: "光线追踪（Ray Tracing）的核心算法是什么？BVH（Bounding Volume Hierarchy）如何加速光线求交？"
chapter: 8
chapter_name: "高级渲染技术与前沿方向"
difficulty: advanced
knowledge_points:
  - id: "08.01"
    name: "光线追踪基础与BVH"
tags: ["ray-tracing", "bvh", "sah", "acceleration-structure"]
---

# Q08.01 光线追踪（Ray Tracing）的核心算法是什么？BVH（Bounding Volume Hierarchy）如何加速光线求交？

**难度：** 🔵 高级
**所属章节：** [高级渲染技术与前沿方向](./index.md)

---

## 🎯 结论

- 光线追踪通过从相机发射光线、与场景几何体求交、递归计算反射/折射来模拟光的传播，BVH（层次包围盒）是加速光线求交的核心数据结构。BVH将场景空间组织为二叉树结构，光线遍历时通过AABB（轴对齐包围盒）测试快速跳过不可能相交的子树，将求交复杂度从O(n)降低到O(log n)，是实时光线追踪得以实现的基础。

## 📐 原理解析

### Whitted-style 递归光线追踪

- 从相机（眼睛）向每个像素发射一条主光线（Primary Ray），找到最近交点
- 在交点处根据材质属性递归发射反射光线（Reflection Ray）和折射光线（Refraction Ray）
- 向光源发射阴影光线（Shadow Ray），判断交点是否在阴影中
- 递归终止条件：达到最大递归深度、光线能量低于阈值、或命中光源
- Whitted-style是经典算法，能精确处理镜面反射和透明折射，但无法处理焦散（Caustics）和漫反射间接光照

### 路径追踪（Path Tracing）与蒙特卡洛积分

- 将渲染方程（Rendering Equation）转化为蒙特卡洛积分问题，通过随机采样估计光照贡献
- 在每个交点处根据BRDF的概率分布随机选择一个方向发射一条光线（而非递归分裂多条）
- 多次采样取平均，随着采样数增加逐渐收敛到正确解（无偏估计）
- 支持漫反射间接光照、焦散、次表面散射等复杂光照现象
- 是电影级离线渲染的标准方法，也是实时光线追踪的理论基础

### BVH（Bounding Volume Hierarchy）加速结构

- BVH将场景中的所有图元（三角形）组织为二叉树，每个节点存储一个包围盒（通常为AABB）
- 构建阶段：自顶向下递归划分——选择最长轴，按中位数或SAH（Surface Area Heuristic）分割图元集
- 遍历阶段：光线从根节点开始，先与节点AABB求交；若不相交则跳过整棵子树；若相交则递归检查子节点
- 叶节点存储实际图元，光线与叶节点内的三角形做精确求交测试
- SAH构建策略：选择分割平面使得 C = C_trav + P_left * N_left * C_isect + P_right * N_right * C_isect 最小，其中P为光线命中概率（与表面积成正比）


## 🛠 工程实践

### BVH构建策略

- SAH（Surface Area Heuristic）是工业界标准的BVH构建方法，质量远优于中位数分割
- 对于静态场景，使用高质量的SAH构建（如binned SAH或spatial splits），构建时间可接受（数百毫秒到数秒）
- 对于动态场景，需要权衡构建质量与构建速度，常用方案包括快速中位数分割、局部Refit（仅更新包围盒不重建树结构）
- Intel的Embree和NVIDIA的OptiX都提供了高度优化的BVH实现，支持SIMD批量光线遍历

### BVH更新策略

- Refit（重新拟合）：不改变树结构，仅自底向上更新每个节点的包围盒，时间复杂度O(n)，适用于物体位移较小的情况
- Rebuild（重建）：完全重新构建BVH，质量最高但开销最大，适用于物体大幅运动或拓扑变化的情况
- 混合策略：对静态物体使用高质量预构建BVH，对动态物体使用快速构建+Refit，通过两级BVH（Top-Level + Bottom-Level）组合
- DXR/Vulkan RT采用两级加速结构：TLAS（Top-Level Acceleration Structure）管理实例变换，BLAS（Bottom-Level Acceleration Structure）存储几何数据，支持动态更新TLAS而保持BLAS不变

### DXR/Vulkan Ray Tracing API实践

- DXR提供了Ray Generation、Intersection、Closest Hit、Miss等Shader阶段，通过Pipeline State Object配置
- Acceleration Structure通过D3D12 BuildRaytracingAccelerationStructure命令构建，支持BuildFlag::ALLOW_UPDATE进行快速更新
- Shader Binding Table（SBT）用于绑定不同几何体的着色器，是实现多材质光线追踪的关键机制
- Vulkan Ray Tracing（VK_KHR_ray_tracing_pipeline）提供类似功能，通过accelerationStructureNV/VK_KHR_acceleration_structure扩展


## ⚠️ 踩坑经验

### BVH构建时间对加载体验的影响

- 高质量SAH构建可能需要数秒时间，严重影响场景加载速度
- 解决方案：异步构建（在加载画面期间后台构建）、流式加载（按区域逐步构建BVH）、预构建并序列化BVH到磁盘
- 移动端设备算力有限，BVH构建时间问题尤为突出，需要使用更激进的简化策略

### 动态场景BVH更新的性能开销

- 每帧重建BVH会消耗大量CPU/GPU时间，可能成为性能瓶颈
- Refit虽然快速，但树结构不随物体运动优化，长时间运行后BVH质量退化，光线遍历效率下降
- 实践建议：对运动幅度小的物体使用Refit，对运动幅度大的物体定期Rebuild；设置质量退化阈值触发重建

### 自相交（Self-Intersection）精度问题

- 光线从表面发射时，由于浮点精度限制，可能立即与自身表面相交，产生黑色噪点（Shadow Acne）
- 解决方案：沿光线方向偏移一个小的epsilon值（通常为1e-3到1e-4），或使用基于法线和光线方向的自适应偏移
- 更稳健的方案是使用IEEE 754浮点数的\
- 函数，计算最小安全偏移量
- 在透明材质的折射计算中，自相交问题更加复杂，需要根据材质折射率调整偏移方向


## 🔮 延伸思考

### 硬件光线追踪核心（RT Core）的架构设计

- NVIDIA RT Core（Turing架构引入）专用于BVH遍历和光线-三角形求交，单时钟周期可完成一次Box Test或Triangle Test
- RT Core采用流水线设计，支持大量光线并行处理，通过SM（Streaming Multiprocessor）调度光线追踪任务
- AMD的Ray Accelerator采用类似设计，集成在Compute Unit中，支持BVH遍历和相交测试
- 硬件RT核心的引入使得实时光线追踪成为可能，但仍需配合降噪算法才能在低采样率下获得可接受画质

### 光线追踪在游戏中的实际应用场景

- 反射效果：水面、金属表面、镜面的实时反射，替代传统Screen Space Reflection（SSR）的局限性
- 软阴影：Area Light的软阴影效果，通过采样光源上的多个点实现自然阴影过渡
- 环境光遮蔽（RTAO）：比SSAO更准确的环境光遮蔽，能正确处理遮蔽关系
- 全局光照（RTGI）：一次或多次弹射的间接光照，为场景提供真实的间接照明
- 透明与折射：玻璃、水等透明材质的折射效果，传统光栅化难以准确模拟

---

[← 返回 高级渲染技术与前沿方向 题目列表](./index.md) | [返回总纲](../00-总纲.md)
