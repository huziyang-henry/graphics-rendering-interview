---
id: Q03.07
title: "虚拟阴影贴图（Virtual Shadow Maps，如UE5的实现）的核心思想是什么？它如何解决传统CSM的分辨率分配不均问题？"
chapter: 3
chapter_name: "阴影技术"
difficulty: expert
knowledge_points:
  - id: "03.04"
    name: "虚拟阴影贴图（Virtual Shadow Maps）"
tags: ["virtual-shadow-map", "ue5", "page-table", "feedback"]
---

# Q03.07 虚拟阴影贴图（Virtual Shadow Maps，如UE5的实现）的核心思想是什么？它如何解决传统CSM的分辨率分配不均问题？

**难度：** 🟣 专家级
**所属章节：** [阴影技术](./index.md)

---

## 🎯 结论

- 虚拟阴影贴图（Virtual Shadow Maps
- VSM从根本上解决了传统CSM的分辨率分配不均问题——不再预先为每个级联分配固定分辨率的Shadow Map，而是让每个屏幕像素“请求”自己所需的Shadow Map精度，实现了真正的按需分辨率分配。

## 📐 原理解析

### Page Table机制

- VSM将整个Shadow Map空间划分为固定大小的Page（UE5中为128x128像素）。每个Page有一个唯一的Page ID，由其在Shadow Map空间中的坐标决定。Page的总数可以非常大（如256x256个Page，对应32768x32768的虚拟Shadow Map分辨率），但实际驻留在显存中的Page数量是有限的。
- Page Table是一个二维纹理，记录每个Page ID对应的物理Page在缓存中的位置。当着色器需要查询某个屏幕像素的阴影时，首先计算该像素在Shadow Map空间中的位置，确定所需的Page ID，然后通过Page Table查找该Page是否在缓存中。如果在缓存中，直接使用缓存的Page进行阴影查询；如果不在缓存中，触发Page Fault，请求渲染该Page。

### Feedback Buffer驱动的按需渲染

- VSM使用Feedback Buffer来实现按需渲染。在每一帧的阴影查询阶段，如果着色器发现所需的Page不在缓存中，它会将Page ID写入Feedback Buffer。Feedback Buffer记录了当前帧所有需要的Page请求。
- 在下一帧（或当前帧的后续Pass），渲染管线根据Feedback Buffer中的请求来渲染缺失的Page。只有被请求的Page才会被渲染，未被请求的区域不消耗任何GPU资源。
- 这种Feedback机制引入了至少一帧的延迟——第一帧请求Page，第二帧才能使用。为了最小化这个延迟的影响，VSM使用上一帧的Page缓存来满足当前帧的大部分请求，只有新出现的可视区域才会触发Page Fault。

### 与传统CSM的对比

传统CSM为每个级联预分配固定分辨率的Shadow Map，无论该级联中的场景内容是否需要这么高的分辨率。例如，一个2048x2048的级联可能覆盖了一大片空旷区域，大部分纹素被浪费。而VSM只为实际可见且需要阴影的区域分配Page，空旷区域不占用任何资源。这意味着VSM可以在不增加总内存开销的前提下，将有效分辨率提高数倍甚至数十倍。


## 🛠 工程实践

### UE5中VSM的实现细节

- Feedback Buffer：使用一个原子计数器缓冲区来收集Page请求。每个屏幕像素对应的Shadow Map Page ID被写入缓冲区，通过原子操作去重。GPU端使用Compute Shader来处理Feedback Buffer并生成渲染指令。
- Page Table：存储在一张纹理中，每个纹素对应一个Page的映射信息（包括物理缓存位置、有效标志等）。Page Table的查找在着色器中通过Texture2D的Load操作完成，开销极低。
- Cache策略：使用LRU（Least Recently Used）缓存替换策略。当缓存已满且需要加载新Page时，淘汰最近最少使用的Page。缓存大小可以通过项目设置来调整，通常限制在128MB-512MB之间。
- 最大Page数量限制：为了控制显存占用，VSM设置了最大驻留Page数量的硬限制。当场景复杂度超过这个限制时，远处的Page会被优先淘汰，可能导致远处阴影质量的下降。

### VSM与Nanite的协同

VSM与UE5的Nanite虚拟几何体系统深度集成。Nanite的GPU Driven渲染管线可以直接生成VSM所需的Shadow Map Page，无需CPU端的Draw Call。这种集成使得VSM可以利用Nanite的LOD系统——远处物体使用低LOD渲染Shadow Map，近处物体使用高LOD，进一步优化了阴影渲染的性能。


## ⚠️ 踩坑经验

### VSM的内存占用

- 虽然VSM是按需分配的，但在复杂场景中，被请求的Page数量可能非常大。例如，一个从高处俯瞰的场景可能同时需要覆盖大范围的Shadow Map Page，导致缓存压力骤增。
- 每个Page的存储格式也会影响内存占用。UE5中每个Page为128x128像素，使用32位浮点深度格式，单个Page约64KB。如果缓存限制为10000个Page，则总内存约640MB。这对显存是一个不小的压力。
- 实践中需要根据目标平台的显存容量来调整缓存大小。在显存受限的平台上（如8GB显存的显卡），可能需要降低Page分辨率或减少最大Page数量。

### Page Fault导致的帧率波动

当摄像机快速移动或场景发生剧烈变化时（如爆炸、建筑物倒塌），大量新的Shadow Map Page需要在同一帧内渲染，导致GPU负载突增，帧率下降。这种帧率波动被称为“Page Fault Spike”。UE5通过以下方式缓解：限制每帧渲染的Page数量（将渲染分散到多帧）；使用上一帧的Page缓存作为降级方案（当新Page未就绪时使用旧数据）；在摄像机快速移动时主动预取可能需要的Page。尽管如此，极端场景下的帧率波动仍是VSM的一个已知问题。


## 🔮 延伸思考

### VSM与Ray-Traced Shadows的对比

- VSM和Ray-Traced Shadows代表了两种不同的阴影技术路线。VSM基于光栅化管线，通过虚拟纹理技术优化分辨率分配，兼容性好，可以在不支持光线追踪的硬件上运行。Ray-Traced Shadows基于光线追踪管线，物理上更准确，支持任意形状的软阴影，但需要硬件支持且性能开销较大。
- 在中短期内，VSM在兼容性和性能方面仍有优势，尤其在大规模开放世界场景中。Ray-Traced Shadows在视觉质量上更胜一筹，适合对画质要求极高的高端应用。两者的融合（如VSM作为Ray-Traced Shadows的降级方案）可能是未来的发展方向。

### VSM在移动端的可行性

VSM在移动端面临多重挑战：显存容量有限（通常4-8GB，且与系统共享）、Feedback Buffer的原子操作在部分移动GPU上性能不佳、Page Table的额外纹理查找增加着色器开销。目前VSM主要面向高端PC和主机平台。在移动端，优化的CSM方案（如2级联+低分辨率+PCF）仍然是更实际的选择。未来随着移动GPU架构的进步（如支持原子操作的Fragment Shader Interlock），VSM在移动端的可行性可能会逐步提高。

---

[← 返回 阴影技术 题目列表](./index.md) | [返回总纲](../00-总纲.md)
