# 02. 光照与着色模型

> 考察候选人对光照原理、着色模型及PBR理论体系的掌握深度

---

## 知识点

| 编号 | 知识点 | 关联题目数 |
|------|--------|-----------|
| 02.01 | 经典光照模型（Phong/Blinn-Phong） | 1 |
| 02.02 | 光照分量（环境光/漫反射/镜面反射） | 1 |
| 02.03 | PBR核心原则与能量守恒 | 1 |
| 02.04 | BRDF与Cook-Torrance模型 | 2 |
| 02.05 | 微表面理论（NDF/Fresnel/Geometry） | 4 |
| 02.06 | IBL（基于图像的光照） | 1 |
| 02.07 | 材质工作流（Metallic-Roughness） | 1 |
| 02.08 | 多层材质与高级着色 | 1 |
---

## 题目列表

### 初级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q02.01 | Phong光照模型和Blinn-Phong光照模型的区别是什么？为什么Blinn-Phong在实践中更常用？ | 经典光照模型（Phong/Blinn-Phong） | [查看](./phong-vs-blinn-phong.md) |
| Q02.02 | 什么是环境光（Ambient）、漫反射（Diffuse）和镜面反射（Specular）？它们各自的物理直觉是什么？ | 光照分量（环境光/漫反射/镜面反射） | [查看](./ambient-diffuse-specular.md) |

### 中级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q02.03 | 请解释PBR（基于物理的渲染）的核心原则。为什么说PBR比传统光照模型更"物理正确"？ | PBR核心原则与能量守恒 | [查看](./pbr-core-principles.md) |
| Q02.04 | 什么是BRDF（双向反射分布函数）？请写出Cook-Torrance BRDF的公式并解释每一项的物理含义。 | BRDF与Cook-Torrance模型, 微表面理论（NDF/Fresnel/Geometry） | [查看](./brdf-cook-torrance.md) |
| Q02.05 | 微表面理论中的法线分布函数（NDF）有哪几种常见实现？GGX和Beckmann的区别是什么？ | 微表面理论（NDF/Fresnel/Geometry） | [查看](./ndf-ggx-vs-beckmann.md) |
| Q02.06 | 菲涅尔效应（Fresnel Effect）在PBR中如何建模？Schlick近似公式的优缺点是什么？ | 微表面理论（NDF/Fresnel/Geometry） | [查看](./fresnel-effect-schlick-approximation.md) |

### 高级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q02.07 | 什么是IBL（基于图像的光照）？如何用辐照度图和预滤波环境贴图实现IBL？ | IBL（基于图像的光照） | [查看](./ibl-irradiance-prefiltered-envmap.md) |
| Q02.08 | 在PBR框架下，如何正确处理Metallic-Roughness工作流中金属和非金属材质的F0值？ | 材质工作流（Metallic-Roughness）, 微表面理论（NDF/Fresnel/Geometry） | [查看](./metallic-roughness-f0-determination.md) |

### 专家级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q02.09 | 在PBR框架下，如何正确处理多层材质（如清漆层Coat、次表面散射SSS）？ | 多层材质与高级着色, BRDF与Cook-Torrance模型 | [查看](./multi-layer-materials-clearcoat-sss.md) |

