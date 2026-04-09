---
id: Q08.03
title: "全局光照（Global Illumination）有哪些经典算法？请比较辐射度方法、光线追踪、光子映射和预计算辐射传输（PRT）的优缺点。"
chapter: 8
chapter_name: "高级渲染技术与前沿方向"
difficulty: advanced
knowledge_points:
  - id: "08.03"
    name: "全局光照算法"
tags: ["gi", "radiosity", "path-tracing", "photon-mapping", "prt"]
---

# Q08.03 全局光照（Global Illumination）有哪些经典算法？请比较辐射度方法、光线追踪、光子映射和预计算辐射传输（PRT）的优缺点。

**难度：** 🔵 高级
**所属章节：** [高级渲染技术与前沿方向](./index.md)

---

## 🎯 结论

- 全局光照（GI）算法按计算时机可分为预计算方法和实时方法两大类。辐射度方法（Radiosity）和预计算辐射传输（PRT）属于预计算方法，在运行时查询预计算结果，适合静态场景；路径追踪和光子映射属于基于采样的方法，其中路径追踪已成为离线渲染的标准，其实时变体（RTGI）正在游戏引擎中快速普及。不同方法在质量、性能、动态性和实现复杂度之间存在根本性的权衡。

## 📐 原理解析

### 辐射度方法（Radiosity）

- 基于有限元方法（Finite Element Method），将场景表面离散化为面片（Patch），计算面片之间的能量传递
- 核心公式： $B_i = E_i + \rho_i \sum_j F_{ij} B_j$ ，其中B为辐射度，E为自发光， $\rho$ 为反射率，F为形状因子（Form Factor）
- 形状因子 $F_{ij}$ 描述面片j发出的能量中到达面片i的比例，仅与几何关系有关，可预计算
- 求解方法：逐步求精（Progressive Refinement）、矩阵求解（Gauss-Seidel迭代）
- 优点：结果与视角无关（View-Independent），预计算一次即可在任意视角查看
- 缺点：只能处理漫反射（Diffuse），无法处理镜面反射和折射；面片离散化导致精度有限

### 预计算辐射传输（Precomputed Radiance Transfer, PRT）

- 将入射光照和BRDF分别用球谐函数（Spherical Harmonics, SH）展开，预计算传输矩阵
- 核心思想：将渲染方程中的积分运算转化为SH系数的矩阵乘法，运行时仅需一次矩阵乘法即可获得每个顶点的出射辐射度
- 支持漫反射和低频镜面反射（通过球面卷积Spherical Convolution），可处理软阴影和漫反射间接光
- 优点：运行时开销极低（仅矩阵乘法），可在移动端实现实时GI
- 缺点：SH的低频特性限制了可表达的光照复杂度（通常只能到 $L=3$ 或 $L=4$ 阶，约16-25个系数），无法表达高频细节如尖锐阴影和高光

### 光子映射（Photon Mapping）

- 两阶段算法：第一阶段从光源发射光子并在场景中弹射，存储到光子图（Photon Map）中；第二阶段从相机发射光线，在交点处查询光子图估计辐射度
- 光子图使用KD-Tree进行最近邻查询，通过核密度估计（Kernel Density Estimation）计算辐射度
- 分为全局光子图（Global Photon Map，记录所有弹射）和焦散光子图（Caustic Photon Map，仅记录第一次弹射）
- 优点：能正确处理焦散（Caustics），这是传统路径追踪难以高效处理的现象；支持参与介质（Participating Media，如云、烟雾）的渲染
- 缺点：光子图的查询引入偏差（Bias），需要仔细调节光子数量和查询半径；实时化困难，但Progressive Photon Mapping等变体有所改善

### 路径追踪（Path Tracing）

- 基于蒙特卡洛积分的无偏渲染算法，通过随机采样路径估计渲染方程的解
- 支持所有类型的光照现象：漫反射、镜面反射、折射、焦散、次表面散射、体积散射
- Bidirectional Path Tracing（BDPT）：从光源和相机同时发射路径，在中间连接，对焦散和室内场景效率更高
- Metropolis Light Transport（MLT）：基于马尔可夫链蒙特卡洛（MCMC）的方法，对困难路径（如焦散）进行重要性采样
- 优点：理论完备，能正确处理所有光照现象，是无偏的（收敛到正确解）
- 缺点：收敛速度慢，特别是对于高维积分（如焦散、间接光照），需要大量采样才能获得低噪声结果


## 🛠 工程实践

### 预计算GI（Lightmap Baking）的工程实践

- Lightmap Baking本质上是辐射度方法的工程化实现：在离线阶段使用路径追踪或辐射度算法计算每个texel的辐照度，存储为Lightmap纹理
- 关键技术：Lightmap的UV布局（确保无重叠、最小化接缝）、Chart的Packing效率、Bake的分辨率设置
- 动态光照处理：Lightmap存储间接光照，直接光照在运行时实时计算（使用Shadow Map或Light Culling）
- 方向性Lightmap（Directional Lightmap）：存储辐照度的球谐系数（通常3个SH基函数），支持间接光照的方向性
- 仍是移动端和部分PC游戏的主流GI方案，Unity的Progressive Lightmapper和UE的Lightmass都是成熟的Baking工具

### 实时光线追踪GI（RTGI）的工程实践

- UE5 Lumen：混合光线追踪和表面缓存实现全动态GI，不依赖预计算
- NVIDIA RTXGI（RTX Global Illumination）：基于世界空间探针（World Space Probe）的实时GI方案，使用DDGI（Dynamic Diffuse Global Illumination）算法
- DDGI算法：在场景中放置3D探针网格，每帧使用光线追踪更新探针的辐照度值，运行时通过三线性插值查询间接光照
- Unity的RTX GI：基于DDGI的实现，与HDRP深度集成，支持动态场景的实时间接光照
- 性能优化：降低探针更新频率（每2-4帧更新一次）、降低光线追踪分辨率、使用AI降噪减少所需采样数


## ⚠️ 踩坑经验

### Lightmap的分辨率和UV布局问题

- Lightmap分辨率不足会导致间接光照出现明显的块状伪影（Blocky Artifacts），特别是大面片上的颜色渐变
- UV接缝处容易出现光照不连续（Seam Artifacts），需要使用边缘扩张（Edge Dilation）和接缝缝合（Seam Stitching）
- Lightmap的UV布局需要为每个物体分配足够的纹理空间，复杂场景的Packing效率直接影响Lightmap的总分辨率和内存占用
- 动态物体（如角色、可移动物体）无法使用Lightmap，需要额外的实时GI方案补充

### PRT的低频光照限制

- 球谐函数的阶数限制（通常 $L=2$ 到 $L=4$ ）决定了PRT只能表达低频光照，无法捕捉尖锐的阴影边缘和高频镜面反射
- 对于点光源或方向光的硬阴影，PRT的结果会显得过度模糊，不适用于需要精确阴影的场景
- PRT假设光照环境是远场的（Distant Lighting），对于近场光源（如室内的台灯）效果不佳
- 增加SH阶数可以提高质量，但系数数量按 $O(L^2)$ 增长，内存和计算开销迅速增加

### 实时GI的动态范围问题

- 实时光线追踪GI在低采样率下容易出现亮度闪烁（Flickering），特别是在间接光照贡献较大的暗区域
- 高动态范围（HDR）场景下，间接光照的亮度可能跨越多个数量级，时域累积时容易出现亮度爆炸（Firefly）
- 解决方案：使用对数空间的累积、亮度裁剪（Luminance Clamping）、自适应采样（在亮度变化大的区域增加采样）
- 探针方案（如DDGI）的探针密度不足时，间接光照的空间细节会丢失，出现漏光（Light Leaking）或光照闪烁（Banding）现象


## 🔮 延伸思考

### Lumen如何实现无需预计算的动态GI

- Lumen的核心创新：Surface Cache（表面缓存）作为动态Lightmap的替代，每帧更新场景表面的辐照度信息
- Surface Cache以屏幕空间分辨率（而非世界空间分辨率）分配资源，天然适应视点变化，避免了Lightmap的全局分辨率问题
- 光线追踪路径从Surface Cache的表面点发射光线收集间接光照，结果写回Surface Cache
- Software Lumen路径使用屏幕空间光线追踪（SDF Ray Marching）作为硬件RT的回退，确保在无RT硬件的平台上也能运行

### GI算法的收敛速度与质量权衡

- 所有基于采样的GI算法都面临收敛速度与质量的根本权衡：高质量需要更多采样，但实时渲染的采样预算极其有限
- 重要性采样（Importance Sampling）是提高收敛速度的关键技术：根据BRDF和光照分布引导采样方向
- 自适应采样（Adaptive Sampling）：在方差大的区域增加采样，在方差小的区域减少采样，优化采样预算的分配
- 多方法混合可能是最终答案：预计算提供基础间接光照，实时RT提供动态修正，AI降噪弥合质量差距

---

[← 返回 高级渲染技术与前沿方向 题目列表](./index.md) | [返回总纲](../00-总纲.md)
