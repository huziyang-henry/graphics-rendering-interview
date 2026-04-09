---
id: Q09.01
title: "假设你需要在一个开放世界游戏中实现一个昼夜循环系统，包括太阳/月亮运动、天空盒变化、动态光照和阴影更新。请描述你的整体架构设计。"
chapter: 9
chapter_name: "综合场景与系统设计"
difficulty: advanced
knowledge_points:
  - id: "09.01"
    name: "昼夜循环与天气系统"
tags: ["day-night", "time-of-day", "sky", "atmosphere", "dynamic-lighting"]
---

# Q09.01 假设你需要在一个开放世界游戏中实现一个昼夜循环系统，包括太阳/月亮运动、天空盒变化、动态光照和阴影更新。请描述你的整体架构设计。

**难度：** 🔵 高级
**所属章节：** [综合场景与系统设计](./index.md)

---

## 🎯 结论

- 昼夜循环系统以Time of Day（时间参数，0-24小时）为核心驱动，通过全局参数统一驱动天空、光照、阴影、后处理等所有视觉子系统。
- 架构分为三层：时间驱动层（Time Manager）→ 参数映射层（Curve/Table）→ 渲染执行层（各渲染子系统读取参数）。
- 关键挑战在于确保所有视觉参数的平滑过渡，避免日夜切换时的视觉跳变。

## 📐 原理解析

### 太阳/月亮运动模型

- 太阳位置由天文模型计算：赤纬角（Declination）取决于季节/日期，时角（Hour Angle）取决于时间，纬度参数化地理位置
- 简化模型：太阳沿椭圆轨道运动，高度角 = f(time, latitude, season)，方位角 = g(time, latitude, season)
- 月亮位置：与太阳位置偏移约12小时，加入月相（朔望月周期）影响月亮亮度

### 天空渲染

- 使用Atmospheric Scattering模型：Rayleigh散射（天空蓝色）+ Mie散射（太阳周围的光晕）
- Preetham模型或Hillaire的Physically-based Sky模型，通过太阳高度角驱动大气密度和颜色变化
- 日出/日落时的大气散射增强（更长的光程导致更红的颜色）

### 光照参数映射

- Directional Light的颜色、强度、方向由Time of Day通过Animation Curve映射
- Ambient Light的颜色和强度同步变化（夜晚偏蓝/紫，白天偏暖白）
- 阴影参数：CSM的级联距离、阴影颜色（夜晚偏蓝）、阴影距离

### 后处理联动

- Tone Mapping的曝光补偿（Exposure Compensation）随时间变化（夜晚降低曝光）
- Color Grading LUT切换（白天/日落/夜晚使用不同的Color Grading）
- Bloom强度和阈值调整（夜晚灯光的Bloom更明显）、Fog颜色和密度变化


## 🛠 工程实践

- UE的Time of Day系统实现：Directional Light + Sky Atmosphere + Exponential Height Fog + Post Process Volume，所有参数通过Curve Editor配置时间曲线。
- 关键参数的Curve配置策略：太阳颜色使用Color Key Curve（关键帧颜色插值）、光照强度使用Float Curve、环境光使用Gradient Curve。
- 天气系统集成：将天气参数（云量、降水、雾密度）作为额外的驱动维度，与Time of Day共同影响最终视觉参数。使用Blend Weight在晴天/阴天/雨天配置之间平滑过渡。
- 性能优化：天空渲染使用低分辨率（1/4或1/2）+ Upsampling；CSM的Shadow Map分辨率在夜晚可以降低（月光阴影质量要求低）。

## ⚠️ 踩坑经验

- 日夜切换时的光照跳变：如果使用离散的关键帧而非连续曲线，会在切换时刻产生明显的亮度/颜色跳变。解决方案是确保所有Curve使用平滑插值（Smooth/TCB）。
- 月亮的阴影实现：月亮光照很弱但仍然需要Directional Light。常见方案是使用第二个Directional Light（仅阴影，强度很低）或在夜晚切换主光源的方向和颜色。
- 星空渲染与大气散射的交互：星星应该在天空足够暗时才出现（太阳高度角 < -10°），且需要考虑大气消光（靠近地平线的星星更暗）。
- 室内外光照过渡：当玩家从室外进入室内时，昼夜循环的影响应该被遮挡。需要使用Light Portal或室内光照探针来处理。

## 🔮 延伸思考

- 程序化天气与昼夜循环的集成：天气系统（云层、降水、雾）与昼夜循环是两个独立的参数维度，它们的组合会产生指数级的状态。使用Multi-dimensional Blend（多维混合）或Precomputed LUT来管理。
- 体积云（Volumetric Cloud）在昼夜循环中的表现：云层的颜色和光照需要与太阳位置实时联动，日出/日落时云层边缘的金色光晕（Silver Lining）是重要的视觉信号。
- 长期的技术趋势：Lumen等动态GI技术使得昼夜循环不再需要预计算Lightmap，大大简化了开放世界昼夜循环的实现。但性能预算仍然是主要挑战。

---

[← 返回 综合场景与系统设计 题目列表](./index.md) | [返回总纲](../00-总纲.md)
