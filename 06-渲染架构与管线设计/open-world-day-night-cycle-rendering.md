---
id: Q06.07
title: "请设计一个支持大世界、动态天气、日夜循环的渲染架构。如何组织渲染Pass、管理资源生命周期？"
chapter: 6
chapter_name: "渲染架构与管线设计"
difficulty: expert
knowledge_points:
  - id: "06.06"
    name: "大世界与多线程渲染架构"
tags: ["open-world", "day-night", "weather", "rendering-architecture"]
---

# Q06.07 请设计一个支持大世界、动态天气、日夜循环的渲染架构。如何组织渲染Pass、管理资源生命周期？

**难度：** 🟣 专家级
**所属章节：** [渲染架构与管线设计](./index.md)

---

## 🎯 结论

- 大世界渲染需要分层架构：近景（CSM + PBR 材质）、中景（LOD + Impostor Billboard）、远景（Skybox + Atmospheric Scattering），配合动态全局参数驱动日夜循环和天气系统。
- 渲染 Pass 的组织需要考虑性能和视觉质量的平衡：Shadow Pass → G-Buffer Pass → Lighting Pass → Sky/Atmosphere Pass → Volumetric Cloud/Fog Pass → Transparent Pass → Post-Process Pass。
- 资源生命周期管理需要支持流式加载/卸载（Streaming）、LOD 过渡和异步资源创建，确保大世界的流畅运行。

## 📐 原理解析

### 大世界渲染的分层架构

- 近景（0-500m）：使用 Cascade Shadow Map（CSM）+ PBR 材质 + 屏幕空间后处理（SSAO、SSR）。物体使用高精度 LOD（LOD0-LOD2）。
- 中景（500m-2km）：使用 LOD + Impostor Billboard（预渲染的物体快照）。阴影使用 Distance Field Shadow 或 Shadow Cascade 的最远级。
- 远景（2km+）：使用 Skybox + Atmospheric Scattering（Rayleigh/Mie 散射）+ Terrain Impostor。不使用逐物体阴影。
- 天空系统：基于物理的 Atmospheric Scattering（Preetham 模型或 Bruneton 模型），根据太阳位置动态计算天空颜色和光照。

### 日夜循环的驱动机制

- 太阳位置由时间（Game Time）驱动，通过天球坐标系计算太阳的方位角和仰角。
- Directional Light 的参数（方向、颜色、强度）根据太阳位置动态更新：正午时太阳光为暖白色、强度最高；黄昏时为橙红色、强度较低；夜晚时以月光为主。
- Sky Shader 根据太阳位置计算大气散射：日出/日落时天空呈现橙红色渐变，正午时为蓝色，夜晚为深蓝/黑色。
- 环境光（Ambient Light）也随时间变化：白天使用基于天空的环境光（IBL），夜晚使用低强度的环境光或人工光源。
- Fog Density 和颜色随时间变化：清晨有薄雾，白天能见度高，夜晚雾气加重。

### 天气系统的实现

- 天气类型：晴天、多云、雨天、雪天、雾天。每种天气类型对应不同的 Particle System 配置、Sky Shader 参数和后处理参数。
- 雨天：使用 GPU Particle System 渲染雨滴（大量小粒子，使用 Compute Shader 更新位置），屏幕空间的雨滴效果（基于法线的扰动），地面的水洼反射。
- 雪天：类似雨天但粒子更大、下落速度更慢，地面使用积雪材质（基于积雪厚度的 Blend）。
- 体积云（Volumetric Cloud）：使用 Ray Marching 在 3D Noise Field 中渲染云层，支持动态光照和阴影投射。
- 天气过渡：不同天气类型之间的切换使用参数插值（Lerp），过渡时间通常为 10-30 秒，避免突兀的视觉变化。


## 🛠 工程实践

### UE5 的 World Partition 架构

- World Partition 将大世界划分为固定大小的 Grid Cell（如 2km × 2km），每个 Cell 包含该区域内的所有 Actor 和资源。
- 基于玩家位置的动态加载/卸载：只加载玩家周围一定范围内的 Cell，远处的 Cell 异步卸载并释放资源。
- 支持 HLOD（Hierarchical LOD）：远处的多个 Actor 合并为一个简化的 HLOD Actor，减少 Draw Call。
- Data Layer 系统：根据游戏进度或任务状态动态加载/卸载特定的 Data Layer，支持非线性的世界探索。

### 动态 Sky Atmosphere 的实现

- 使用 Rayleigh 散射和 Mie 散射的物理模型计算天空颜色。Rayleigh 散射导致短波长光（蓝）散射更强，Mie 散射导致太阳周围的光晕效果。
- LUT（Look-Up Table）预计算：将大气散射的积分计算预计算为 2D/3D LUT，在运行时通过采样 LUT 获取天空颜色，避免逐像素的积分计算。
- 体积云使用 3D Perlin-Worley Noise 定义云的密度场，Ray Marching 沿视线方向步进，在每个步进点采样密度并计算光照。
- UE5 的 Sky Atmosphere 组件提供了基于物理的大气散射实现，支持地面到太空的连续过渡。

### 天气过渡的 Blend 策略

- 使用一个全局的 Weather Blend Factor（0-1）控制两种天气之间的过渡。所有天气相关的参数（Particle Density、Sky Color、Fog Density 等）都根据 Blend Factor 进行插值。
- Particle System 的过渡：旧天气的粒子逐渐消失（减少发射率、增大透明度），新天气的粒子逐渐出现。
- Sky Shader 的过渡：直接插值两种天气的 Sky LUT 参数，或使用 Cross-Fade 混合两种天空渲染结果。
- 地面材质的过渡：雨天时地面的湿润程度（Wetness）逐渐增加，影响 PBR 材质的 Roughness 和 F0 参数。


## ⚠️ 踩坑经验

### 日夜切换时的光照跳变

- 太阳位置的变化可能导致 Directional Light 的方向突然改变，使得阴影方向跳变。解决方案：使用平滑的时间插值和阴影 Cascade 的渐变过渡。
- 环境光的变化可能导致场景整体亮度突然改变。解决方案：使用 Adaptation（类似人眼的瞳孔调节）系统，逐渐调整曝光参数。
- Sky Shader 在日出/日落时可能出现颜色不自然（如天空突然变亮或变暗）。解决方案：在太阳接近地平线时使用额外的渐变参数控制过渡。

### 大世界的 Streaming 和 LOD 管理

- Streaming 的延迟可能导致玩家移动到新区域时出现物体突然出现（Pop-in）或纹理模糊（Mip Streaming 未完成）的问题。
- 解决方案：预加载玩家前进方向上的 Cell（基于移动方向和速度预测），使用异步加载避免主线程卡顿。
- LOD 切换时可能出现几何跳变（ LOD Pop ）：使用 LOD Dithering（基于距离的透明度抖动）或 CLOD（Continuous LOD）平滑过渡。
- HLOD 的生成需要离线预处理，对于频繁修改的场景需要自动化工具链。

### 天气效果的 GPU 开销

- 体积云的 Ray Marching 是 GPU 密集型操作：每个像素需要 64-128 步 Ray Marching，每步需要采样 3D Noise Texture 和计算光照。在 1080p 下可能占用 5-10ms 的 GPU 时间。
- 雨天/雪天的大量粒子更新（Compute Shader）和渲染（Geometry Shader 或 Instanced Drawing）也会增加 GPU 开销。
- 优化方案：降低体积云的 Ray Marching 步数（使用 Temporal Accumulation 补偿）、使用 LOD 减少远处粒子的密度、使用 Compute Shader Culling 剔除屏幕外的粒子。


## 🔮 延伸思考

### 程序化大世界生成与渲染的协同

- 程序化生成（Procedural Generation）与渲染架构的协同：程序化生成的世界需要支持动态的 LOD 和 Streaming，渲染架构需要能够处理运行时生成的几何体和材质。
- 使用 GPU Driven Rendering 处理程序化生成的大量物体：Compute Shader 执行 Culling 和 LOD 选择，Indirect Draw 执行渲染。
- 程序化生成的地形可以使用 Virtual Texture（虚拟纹理）管理超大分辨率的地面纹理，根据视点动态加载/卸载纹理 Tile。

### 云层体积渲染的最新进展

- 基于 Neural Radiance Field（NeRF）的云层渲染：使用神经网络学习云的外观和光照，可以在低计算开销下生成高质量的云层图像。目前主要用于离线渲染，实时性能仍在优化中。
- 基于 Residual Ratio Tracking 的体积云阴影：更精确地计算体积云对地面的阴影投射，减少 Ray Marching 的步进次数。
- Horizon-based Ambient Occlusion for Volumetric Clouds：在体积云内部计算环境光遮蔽，增加云层的立体感和层次感。
- UE5 的体积云系统已经可以在实时游戏中提供电影级的云层渲染效果，结合 Nanite 和 Lumen 实现完整的动态天气和光照。

---

[← 返回 渲染架构与管线设计 题目列表](./index.md) | [返回总纲](../00-总纲.md)
