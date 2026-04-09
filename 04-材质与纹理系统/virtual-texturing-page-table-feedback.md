---
id: Q04.06
title: "虚拟纹理（Virtual Texturing）的原理是什么？它如何解决超大纹理的显存和带宽问题？Page Table和Feedback机制如何工作？"
chapter: 4
chapter_name: "材质与纹理系统"
difficulty: expert
knowledge_points:
  - id: "04.05"
    name: "虚拟纹理（Virtual Texturing）"
tags: ["virtual-texturing", "page-table", "feedback", "vt"]
---

# Q04.06 虚拟纹理（Virtual Texturing）的原理是什么？它如何解决超大纹理的显存和带宽问题？Page Table和Feedback机制如何工作？

**难度：** 🟣 专家级
**所属章节：** [材质与纹理系统](./index.md)

---

## 🎯 结论

- 虚拟纹理（Virtual Texturing
- VT通过Page Table（页表）将虚拟纹理地址映射到物理Page Cache地址，通过Feedback机制（在GPU端记录缺页请求，CPU端异步加载）实现按需加载。核心思想借鉴了操作系统的虚拟内存管理。

## 📐 原理解析

### 纹理分页（Paging）机制

- 超大纹理（如16K x 16K或更大）被分割为固定大小的Page，常见Page尺寸为128x128或256x256像素。
- 每个Page在虚拟纹理空间中有唯一的地址（Page Address），由（Mipmap Level, Page X, Page Y）三元组确定。
- 显存中维护一个固定大小的Page Cache（页缓存），存储当前已加载的Page。Cache大小通常为128MB-512MB，远小于完整虚拟纹理的大小。
- 当渲染时需要访问的Page不在Cache中时，触发\
- （Page Fault），需要从磁盘加载该Page到Cache中。

### Page Table（页表）

- Page Table是一个2D纹理（或Texture Buffer），存储每个虚拟Page到物理Cache Slot的映射关系。
- 每个Page Table条目包含：物理Cache中的Slot索引、该Page的Mipmap层级、有效标志位。
- 在渲染时，片元着色器首先根据UV坐标计算虚拟Page地址，然后查询Page Table获取物理Cache地址，最后从Cache中采样实际纹理数据。
- Page Table本身需要被GPU采样，因此它也占用一定的显存和带宽，但远小于完整纹理。

### Feedback机制

- GPU端：在片元着色器中（通常通过一个单独的Pass或Compute Shader），检查每个像素需要的Page是否在Cache中。如果不在，将该Page的地址写入Feedback Buffer。
- CPU端：每帧读取Feedback Buffer，收集所有缺页请求，按优先级排序（近处优先、屏幕中心优先），异步从磁盘加载这些Page到Page Cache中，并更新Page Table。
- 加载延迟：从缺页请求到Page可用通常需要1-3帧的延迟。在此期间，未加载的Page使用上一帧的数据或低层级Mipmap的Page作为降级显示。
- Feedback Buffer通常使用原子操作（Atomic Counter）或Append Buffer来避免线程冲突，确保每个Page只被请求一次。


## 🛠 工程实践

### UE4/5的Virtual Texture实现细节

- UE4.22引入了Virtual Texturing支持，UE5中VT已成为大纹理的标准处理方式。
- UE的VT实现使用128x128的Page尺寸，支持每材质最多4层虚拟纹理（如Albedo + Normal + Roughness + AO可以打包为一个VT层）。
- Runtime Virtual Texture（RVT）：UE5特有的功能，允许在运行时动态生成虚拟纹理（如全局光照、阴影、SSR结果烘焙到VT中），供后续Pass采样。
- VT与材质系统的集成：材质编辑器中可以指定纹理为Virtual Texture，引擎自动处理VT的采样和回退逻辑。

### Page Cache的LRU策略

- Page Cache使用LRU（Least Recently Used）或其变种策略管理：当Cache满时，淘汰最久未被访问的Page。
- 改进的LRU策略考虑Page的Mipmap层级——高层级（低分辨率）的Page加载成本低，可以优先淘汰；低层级（高分辨率）的Page加载成本高，应尽量保留。
- Page的引用计数：正在被渲染使用的Page不能被淘汰，即使它是最久未使用的。需要维护一个\
- 集合来标记正在使用的Page。
- 预加载策略：根据相机运动方向预测即将需要的Page，提前加载以减少可见的Pop-in。

### Mipmapping在VT中的处理

VT中的Mipmap处理与普通纹理不同。每个Mipmap层级被独立分页，高层级（低分辨率）的Page覆盖更大的虚拟纹理区域。当低层级的Page未加载时，自动回退到高层级的Page（相当于使用更模糊的mipmap）。这种自然的回退机制确保了纹理始终可用，即使在高负载场景下也不会出现黑色或错误的纹理。


## ⚠️ 踩坑经验

### VT的初始加载延迟（Hitch）

当场景首次加载或相机快速移动到新区域时，大量Page同时缺页，导致CPU需要在一帧内处理大量IO请求，造成明显的卡顿（Hitch）。解决方案：实现预加载机制（在场景加载时预加载可见区域的Page）、限制每帧的Page加载数量（平滑加载过程）、使用低分辨率Mipmap作为即时回退。

### Page Cache不足导致的频繁换页（Pop-in）

如果Page Cache大小不足以容纳当前可见的所有Page，系统会不断淘汰和重新加载Page，导致纹理在清晰和模糊之间反复切换（Pop-in）。这通常发生在场景复杂度突然增加或相机快速旋转时。解决方案：增大Page Cache预算（在显存允许的范围内）、优化纹理分辨率（避免不必要的超大纹理）、实现基于重要性的Page保留策略。

### VT与材质系统的集成复杂度

VT要求材质系统支持特殊的采样方式——不能使用标准的texture()函数，必须通过VT专用的采样函数（先查Page Table，再从Cache采样）。这意味着所有使用VT的Shader都需要修改，增加了代码复杂度。此外，VT与通道打包（Channel Packing）的兼容性也需要特别处理——打包后的纹理需要作为整体进行分页，不能单独加载某个通道。


## 🔮 延伸思考

### VT与Nanite的协同工作

在UE5中，VT和Nanite（虚拟几何体）是两个互补的"虚拟化"技术。Nanite解决了几何体精度的问题，VT解决了纹理精度的问题。两者结合使得使用电影级精度的资产（数百万面的模型 + 16K纹理）成为可能。Nanite的Cluster Culling和VT的Page Culling可以协同工作——只有通过Nanite可见性测试的几何体才会触发VT的Page请求，避免了为不可见几何体加载纹理Page的浪费。

### VT在开放世界中的必要性

在开放世界游戏中，地形纹理通常需要极高的分辨率（如8K-16K）以避免近距离观察时的模糊。如果使用传统纹理加载方式，一张16K RGBA8纹理就需要1GB显存，而一个大型开放世界可能有数十张这样的纹理。VT使得只加载玩家周围可见区域的纹理Page成为可能，将显存占用从GB级降低到MB级。随着游戏世界规模的增大和纹理分辨率的提高，VT正在从"可选优化"变为"必需技术"。

---

[← 返回 材质与纹理系统 题目列表](./index.md) | [返回总纲](../00-总纲.md)
