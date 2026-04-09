# 08. 高级渲染技术与前沿方向

> 考察候选人对行业前沿技术的跟踪和理解深度

---

## 知识点

| 编号 | 知识点 | 关联题目数 |
|------|--------|-----------|
| 08.01 | 光线追踪基础与BVH | 1 |
| 08.02 | 实时光线追踪降噪（DLSS/FSR） | 1 |
| 08.03 | 全局光照算法 | 1 |
| 08.04 | UE5渲染架构（Lumen/Nanite） | 2 |
| 08.05 | 可变速率着色（VRS）与VR优化 | 1 |
| 08.06 | 神经渲染（NeRF/3DGS） | 1 |
---

## 题目列表

### 高级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q08.01 | 光线追踪（Ray Tracing）的核心算法是什么？BVH（Bounding Volume Hierarchy）如何加速光线求交？ | 光线追踪基础与BVH | [查看](./ray-tracing-bvh-acceleration.md) |
| Q08.03 | 全局光照（Global Illumination）有哪些经典算法？请比较辐射度方法、光线追踪、光子映射和预计算辐射传输（PRT）的优缺点。 | 全局光照算法 | [查看](./global-illumination-algorithms-comparison.md) |

### 专家级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q08.02 | 实时光线追踪中，Denoising（降噪）为什么是必要的？常见的降噪方法（如NVIDIA DLSS、AMD FSR 3）的原理是什么？ | 实时光线追踪降噪（DLSS/FSR） | [查看](./real-time-ray-tracing-denoising-dlss-fsr.md) |
| Q08.04 | Lumen（UE5的全局光照系统）的核心设计思想是什么？它如何实现无需预计算的动态GI？ | UE5渲染架构（Lumen/Nanite） | [查看](./lumen-ue5-global-illumination-surface-cache.md) |
| Q08.05 | Nanite（UE5的虚拟几何体系统）如何实现海量三角形的实时渲染？请讨论其LOD选择、聚类和软件光栅化的设计。 | UE5渲染架构（Lumen/Nanite） | [查看](./nanite-ue5-virtual-geometry-lod-software-raster.md) |
| Q08.06 | 什么是可变速率着色（Variable Rate Shading, VRS）？它如何与眼球追踪技术结合以提升VR渲染性能？ | 可变速率着色（VRS）与VR优化 | [查看](./variable-rate-shading-vr-foveated-rendering.md) |
| Q08.07 | 神经渲染（Neural Rendering）在游戏引擎中有哪些应用前景？请讨论NeRF和3D Gaussian Splatting的实时化挑战。 | 神经渲染（NeRF/3DGS） | [查看](./neural-rendering-nerf-3d-gaussian-splatting.md) |

