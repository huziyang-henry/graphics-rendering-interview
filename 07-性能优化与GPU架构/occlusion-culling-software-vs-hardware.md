---
id: Q07.04
title: "什么是Occlusion Culling（遮挡剔除）？请比较软件遮挡剔除和硬件遮挡查询（Occlusion Query）的优缺点。"
chapter: 7
chapter_name: "性能优化与GPU架构"
difficulty: intermediate
knowledge_points:
  - id: "07.02"
    name: "剔除技术（视锥体/遮挡/距离）"
tags: ["occlusion-culling", "hiz", "occlusion-query"]
---

# Q07.04 什么是Occlusion Culling（遮挡剔除）？请比较软件遮挡剔除和硬件遮挡查询（Occlusion Query）的优缺点。

**难度：** 🟡 中级
**所属章节：** [性能优化与GPU架构](./index.md)

---

## 🎯 结论

- 遮挡剔除（Occlusion Culling）是一种剔除被不透明物体遮挡的不可见物体的技术，避免对不可见物体执行昂贵的Vertex Shader和Fragment Shader计算。软件遮挡剔除基于预计算或启发式算法在CPU端执行，硬件遮挡查询（Occlusion Query）利用GPU的实际渲染结果判断物体可见性。两者各有优劣，现代引擎通常组合使用。

## 📐 原理解析

### 软件遮挡剔除方法

- BSP/PVS（Potentially Visible Set）：在构建阶段预计算每个区域可能看到的区域集合，运行时根据摄像机位置查表确定可见集合。Quake系列经典方法，适合室内场景
- Portal系统：预计算房间之间的Portal（门/窗），运行时通过Portal之间的可见性传递确定可见房间。适合高度分隔的室内环境
- 基于深度缓冲的HiZ（Hierarchical Z-Buffer）剔除：在CPU端维护一个低分辨率的深度层次结构（Mip Chain），用物体的包围盒与HiZ比较判断遮挡关系
- 基于BVH/Octree的场景空间结构：结合视锥剔除后，对剩余物体进行基于空间结构的遮挡判定

### 硬件遮挡查询（Occlusion Query）

- CPU提交一个Occlusion Query，将物体的包围盒（Bounding Box）渲染到深度缓冲
- GPU执行渲染后，返回实际通过深度测试的像素数量（Samples Passed）
- CPU根据返回的像素数判断物体是否可见——如果像素数为0或低于阈值，则剔除该物体
- 关键问题：GPU执行是异步的，CPU获取Query结果需要等待GPU完成，引入2-3帧的延迟


## 🛠 工程实践

### UE的遮挡剔除系统

- 预计算模式：基于Cell-Portal系统，在构建阶段计算每个Cell的可见性信息，运行时查表
- 运行时模式：基于软件光栅化的HiZ剔除，每帧在CPU端维护深度层次结构
- UE5的Nanite引入了基于软件光栅化的遮挡剔除，在Compute Shader中执行高精度遮挡判定

### Unity的遮挡剔除

- 基于Bake的遮挡剔除：在编辑器中烘焙遮挡数据（基于Cell-Portal方法），运行时使用烘焙数据
- 运行时参数调优：Cell Size影响剔除精度和内存开销，较小的Cell Size提供更精确的剔除但增加内存
- Unity 2022+引入了基于GPU的遮挡查询作为补充手段


## ⚠️ 踩坑经验

### Occlusion Query的GPU-CPU同步延迟

Occlusion Query最大的问题是GPU-CPU同步延迟。CPU提交Query后需要等待GPU完成渲染才能获取结果，这个等待如果同步进行会导致CPU Stall（流水线停顿），如果异步进行则结果延迟2-3帧。延迟意味着剔除决策基于过时的摄像机位置，快速转动的摄像机可能导致物体'闪烁'（应该可见但被错误剔除）。解决方案包括空间分区预测、保守包围盒、多帧结果平滑等。

### 软件遮挡剔除的预计算时间和内存开销

Bake-based的遮挡剔除需要较长的预计算时间（大型场景可能需要数分钟到数小时），且烘焙数据占用额外内存。每次场景修改都需要重新烘焙，影响开发迭代速度。此外，动态物体（如可移动的门、可破坏的墙壁）无法被预计算的遮挡数据覆盖，需要额外的运行时遮挡处理。


## 🔮 延伸思考

### GPU Driven Occlusion Culling

现代GPU Driven渲染管线中，遮挡剔除在Compute Shader中执行：使用上一帧或当前帧的深度缓冲生成HiZ Mip Chain，然后在Compute Shader中对场景BVH进行遍历，利用HiZ进行节点级别的遮挡判定。这种方法完全在GPU端执行，没有GPU-CPU同步延迟问题，且可以利用GPU的大规模并行能力高效处理大量物体。

### Nanite的软件光栅化遮挡剔除

UE5的Nanite使用专用的软件光栅化器（运行在Compute Shader中）进行高精度的遮挡剔除。它将场景几何体光栅化为低分辨率的深度缓冲，用于后续Pass的可见性判定。这种方法比传统的HiZ剔除更精确，因为它可以处理任意复杂的几何体，而不仅仅是包围盒。

---

[← 返回 性能优化与GPU架构 题目列表](./index.md) | [返回总纲](../00-总纲.md)
