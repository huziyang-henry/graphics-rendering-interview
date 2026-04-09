---
id: Q05.07
title: "请设计一个完整的后处理管线，考虑Pass的执行顺序、中间RT的精度要求和带宽优化。"
chapter: 5
chapter_name: "后处理效果"
difficulty: expert
knowledge_points:
  - id: "05.05"
    name: "后处理管线架构设计"
tags: ["post-process", "pipeline", "render-target", "bandwidth"]
---

# Q05.07 请设计一个完整的后处理管线，考虑Pass的执行顺序、中间RT的精度要求和带宽优化。

**难度：** 🟣 专家级
**所属章节：** [后处理效果](./index.md)

---

## 🎯 结论

- 后处理管线的设计需要在视觉质量和性能之间取得平衡，核心考虑因素包括Pass的执行顺序（确保每个Pass在正确的数据基础上工作）、中间RT的精度选择（在精度和带宽之间权衡）以及计算优化策略（降采样、Pass合并、异步计算）。一个精心设计的后处理管线可以在不牺牲视觉质量的前提下显著降低GPU开销。

## 📐 原理解析

### 典型后处理Pass顺序

- 1. TAA（Temporal Anti-Aliasing）：需要Motion Vector和当前帧Color Buffer，输出为抗锯齿后的Color Buffer。TAA通常是最早的后处理Pass，因为它需要原始渲染结果
- 2. Bloom：从HDR Color Buffer中提取亮部，经过多级降采样和高斯模糊后叠加回原图
- 3. SSAO/SSR：屏幕空间效果，需要深度缓冲和法线缓冲。通常在Bloom之后执行，因为它们需要清晰的深度信息
- 4. DOF（Depth of Field）：基于深度缓冲计算CoC，执行散景模糊。需要在SSAO/SSR之后，因为它会模糊整个画面
- 5. Tone Mapping：将HDR值映射到LDR范围。必须在所有使用HDR值的Pass之后执行
- 6. Gamma Correction：从线性空间转换到Gamma空间（sRGB编码）。必须在Tone Mapping之后
- 7. Color Grading：使用LUT进行颜色调整。在Gamma Correction之后执行，此时颜色值在[0,1]范围内
- 8. FXAA/SMAA：最终的空间抗锯齿Pass，处理TAA可能遗漏的残余锯齿

### 中间RT的精度要求

- HDR阶段（TAA到Tone Mapping之前）：需要浮点RT。R11G11B10_FLOAT（无Alpha，节省带宽）或RGBA16_FLOAT（有Alpha，完整精度）
- R11G11B10_FLOAT的精度：每个通道分别有11/11/10位指数+尾数，足以表示[0, +inf)范围的HDR值，且比RGBA16_FLOAT节省25%的带宽
- LDR阶段（Tone Mapping之后）：可以使用R8G8B8A8_UNORM格式，此时颜色值已在[0,1]范围内
- 深度相关Pass（SSAO、SSR、DOF）：需要线性深度缓冲，通常使用R16_FLOAT或R32_FLOAT格式


## 🛠 工程实践

### Pass合并优化

- Bloom + Tone Mapping合并：将Bloom的最终叠加和Tone Mapping合并到一个Pass中，减少一次全屏RT的读写
- Gamma Correction + Color Grading合并：Gamma编码和LUT查找可以在同一个Fragment Shader中完成
- SSAO + Blur合并：将SSAO的计算和降噪模糊合并为一个Pass，利用Shared Memory（Compute Shader）或双Pass合并
- UE5中大量使用了Pass合并技术，将原本需要10+个Pass的后处理管线压缩到5~7个Pass

### 降采样策略

- Bloom：逐级降采样到1/2、1/4、1/8、1/16，在低分辨率下执行模糊
- SSAO：在1/2分辨率下计算，上采样回全分辨率
- DOF：大CoC区域在1/4分辨率下模糊，小CoC区域在全分辨率下处理
- 降采样时使用高质量的滤波器（如13-tap Lanczos）而非简单的Box Filter，避免降采样引入的锯齿和混叠

### 异步计算的应用

- 在支持异步计算的硬件上（PS4/PS5、Xbox、PC），可以将SSAO计算放到异步计算队列中，与主渲染队列的光照计算并行执行
- Bloom的降采样Pass也可以异步执行，因为它只依赖HDR Color Buffer，不需要等待所有光照计算完成
- 需要注意：异步计算需要仔细管理GPU资源的同步，避免数据竞争。通常使用Fence或Event来确保数据依赖的正确性


## ⚠️ 踩坑经验

### RT精度不足导致的色带（Banding）

- 当使用R11G11B10_FLOAT格式时，在暗部区域（低亮度值）的精度可能不足以表示平滑的渐变，导致可见的色带
- 特别是在Tone Mapping之后的暗部区域，如果直接输出到R8G8B8A8_UNORM，8位精度在暗部只有很少的可区分色阶
- 解决方案：在最终输出前添加Dithering（有序抖动或随机抖动），将色带打散为肉眼不易察觉的噪点
- 更高质量的方案：使用计算着色器实现蓝噪声Dithering，视觉效果最佳

### Pass顺序错误导致的视觉瑕疵

- 常见错误1：在Tone Mapping之前做Color Grading。此时颜色值仍在HDR范围，LUT的[0,1]采样范围无法覆盖HDR值
- 常见错误2：在Bloom之前做Tone Mapping。此时高亮信息已经被压缩，Bloom无法提取到足够的亮部
- 常见错误3：在DOF之前做SSR。DOF的模糊会破坏SSR的反射细节
- 调试建议：使用RenderDoc等工具逐步检查每个Pass的输入输出，确认数据流的正确性

### 后处理对延迟（Frame Time）的影响

- 后处理Pass通常在帧末尾执行，它们的执行时间直接增加帧延迟。在60fps的目标下，整个后处理管线通常需要控制在3~5ms以内
- 带宽是后处理的主要瓶颈而非计算。全屏Pass的带宽消耗 = RT宽度 × 高度 × 每像素字节数 × Pass数量
- 在4K分辨率下，一次全屏Pass读写RGBA16_FLOAT就需要约128MB的带宽。10个Pass就是1.28GB，远超GPU的带宽能力
- 解决方案：降采样（减少像素数）、Pass合并（减少Pass数）、使用Compute Shader（更灵活的Cache控制）


## 🔮 延伸思考

### Compute-based后处理管线的优势

- 使用Compute Shader替代Fragment Shader执行后处理可以获得更灵活的内存访问模式：Shared Memory（LDS）可以实现Tile-based的局部数据共享，大幅减少全局内存访问
- Compute Shader可以精确控制Workgroup的尺寸和调度，避免Fragment Shader中不可控的内存访问模式
- 典型的Compute优化：将SSAO的采样和模糊合并到一个Compute Pass中，利用Shared Memory在Workgroup内共享深度和法线数据
- UE5的后处理管线大量使用Compute Shader，特别是在次世代平台上

### 移动端后处理的精简策略

- 移动端GPU的带宽和计算能力远低于桌面端，后处理管线需要大幅精简
- 推荐的移动端后处理管线：TAA（简化版）→ Bloom（1~2级降采样）→ Tone Mapping + Gamma + Color Grading（合并为一个Pass）→ FXAA
- 可以完全跳过的Pass：SSAO（移动端通常用静态AO贴图替代）、SSR（使用环境贴图替代）、DOF（使用简单的径向模糊替代）
- 移动端特别需要注意的：避免使用RGBA16_FLOAT格式（部分Mali GPU不支持），使用R11G11B10_FLOAT或半精度浮点替代

---

[← 返回 后处理效果 题目列表](./index.md) | [返回总纲](../00-总纲.md)
