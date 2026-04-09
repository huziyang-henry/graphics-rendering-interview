---
id: Q08.02
title: "实时光线追踪中，Denoising（降噪）为什么是必要的？常见的降噪方法（如NVIDIA DLSS、AMD FSR 3）的原理是什么？"
chapter: 8
chapter_name: "高级渲染技术与前沿方向"
difficulty: expert
knowledge_points:
  - id: "08.02"
    name: "实时光线追踪降噪（DLSS/FSR）"
tags: ["denoising", "dlss", "fsr", "super-resolution", "temporal"]
---

# Q08.02 实时光线追踪中，Denoising（降噪）为什么是必要的？常见的降噪方法（如NVIDIA DLSS、AMD FSR 3）的原理是什么？

**难度：** 🟣 专家级
**所属章节：** [高级渲染技术与前沿方向](./index.md)

---

## 🎯 结论

- 实时光线追踪由于计算预算有限，每像素只能发射极少量的光线（通常1-4 samples per pixel，spp），导致蒙特卡洛积分的方差极大，图像充满噪点。降噪（Denoising）通过时域累积、空间滤波或AI推理，将低采样率的噪点图像恢复为干净的高质量图像，是实时光线追踪能否落地的关键技术。NVIDIA DLSS和AMD FSR 3分别代表了AI驱动和算法驱动的两种降噪/超分辨率路线。

## 📐 原理解析

### 时域降噪（Temporal Denoising）

- 核心思想：利用运动向量（Motion Vector）将当前帧的光线追踪结果与历史帧的结果对齐并累积
- 通过多帧累积，等效采样数从1 spp增加到数十甚至数百spp，大幅降低噪声
- 关键技术：运动向量精度（需要亚像素级精度）、历史帧的权重衰减（越旧的帧权重越低）、重投影失败检测（物体遮挡/露出时的处理）
- 代表实现：NVIDIA RTX Denoiser（早期版本）、UE5的Temporal Super Resolution

### 空间降噪（Spatial Denoising）

- 核心思想：利用图像的空间相关性，对邻域像素进行加权滤波
- 挑战：简单的均值/高斯模糊会丢失细节，需要边缘感知（Edge-Aware）滤波器
- 常用方法：双边滤波（Bilateral Filter，基于颜色和法线差异加权）、非局部均值（Non-Local Means）、引导滤波（Guided Filter，使用G-Buffer辅助）
- A-SVGF（Adaptive Spatial Variance-Guided Filter）：结合方差估计和边缘检测的自适应滤波器，是学术界标杆

### AI降噪（NVIDIA DLSS Super Resolution）

- DLSS使用卷积神经网络（CNN）将低分辨率、高噪声的光线追踪输入转换为高分辨率、低噪声的输出
- 网络输入：低分辨率颜色缓冲、运动向量、深度缓冲、法线缓冲、曝光信息、历史帧信息（多帧时域输入）
- 网络架构：基于U-Net的编码器-解码器结构，结合注意力机制和残差连接
- 训练数据：使用极高采样率（数千spp）的离线渲染结果作为Ground Truth，低采样率图像作为输入，通过监督学习训练
- DLSS 3.5引入了Ray Reconstruction，直接对光线追踪的原始信号进行AI重建，替代传统的手动降噪管线

### AMD FSR 3（FidelityFX Super Resolution 3）

- FSR 3的核心是Frame Generation（帧生成）技术，通过光学流（Optical Flow）在两个渲染帧之间插值生成中间帧
- 时域放大（Temporal Upscaling）：类似DLSS的时域超分辨率，但使用算法而非AI，基于运动向量重投影和反Jitter累积
- 光学流计算：使用专门的Compute Shader分析连续帧之间的像素运动，生成密集光流场
- 帧生成流程：渲染帧N → 计算光流 → 插值生成帧N+0.5 → 渲染帧N+1 → 插值生成帧N+1.5，实现帧率翻倍
- 与DLSS的区别：FSR 3是纯算法方案，不依赖专用AI硬件，跨平台兼容性更好，但画质通常略逊于DLSS


## 🛠 工程实践

### DLSS Super Resolution集成实践

- 引擎集成步骤：初始化DLSS模块 → 创建低分辨率RT目标 → 设置Jitter Pattern → 执行光线追踪 → 调用DLSS推理 → 输出高分辨率图像
- 需要提供的G-Buffer：Motion Vector（运动向量）、Depth（深度）、Normal（法线）、Exposure（曝光值）
- Jitter Pattern：每帧对像素采样位置施加亚像素偏移（如Halton序列），确保时域累积时覆盖不同的子像素位置
- DLSS Quality Mode：提供Quality（1.5x）、Balanced（1.7x）、Performance（2x）、Ultra Performance（3x）等预设，数字表示分辨率缩放因子

### FSR 3 Frame Generation集成实践

- 引擎需要实现UI分帧（UI Frame Interpolation）：UI元素不参与帧生成，需要单独合成，避免UI模糊
- 需要实现帧Pacing机制：确保渲染帧和生成帧的展示节奏稳定，避免帧时间不均匀导致的卡顿感
- 光学流质量对帧生成效果至关重要：快速运动物体、遮挡/露出区域容易出现伪影
- FSR 3需要配套使用帧生成和超分辨率才能获得最佳效果，单独使用帧生成时输入分辨率需要足够高

### 降噪质量评估与调优

- 评估指标：SSIM（结构相似性）、PSNR（峰值信噪比）、LPIPS（学习感知图像块相似度，更符合人眼感知）
- 常见问题检查清单：鬼影（Ghosting，运动物体残影）、闪烁（Flickering，时域不稳定）、拖影（Smearing，运动模糊过度）、细节丢失（Detail Loss，纹理被过度平滑）
- 调优建议：确保运动向量精度、调整历史帧累积权重、检查G-Buffer的精度和格式


## ⚠️ 踩坑经验

### 降噪导致的细节丢失和鬼影（Ghosting）

- 时域降噪的最大问题是鬼影：当物体快速运动或场景出现遮挡变化时，历史帧的信息不再有效，但降噪器仍会使用这些过时信息
- 表现：运动物体后面出现半透明的残影，严重影响视觉质量
- 解决方案：使用更严格的运动向量验证（检查深度差异）、引入遮挡检测（Occlusion Culling）、设置历史帧的最大有效期
- DLSS通过端到端学习自动学习何时丢弃历史帧信息，效果优于手动设计的时域滤波器

### DLSS的Training Data Bias问题

- DLSS的训练数据由NVIDIA使用特定引擎和场景生成，可能对某些类型的场景（如极细线条、高频纹理、UI元素）处理不佳
- 特定游戏风格（如卡通渲染、像素风）可能不在训练分布内，降噪效果可能不理想
- DLSS版本迭代（1.0→2.0→3.0→3.5）每次都改进了训练数据和网络架构，老版本的问题在新版本中可能已修复
- 开发者无法自定义DLSS的降噪行为，遇到边缘情况时缺乏调优手段

### FSR在不同场景下的质量差异

- FSR在慢速运动、高细节场景下表现良好，但在快速运动、低纹理场景下容易出现明显的伪影
- 光学流在重复纹理区域（如草地、水面）容易计算错误，导致帧生成时出现撕裂或错位
- FSR的算法降噪在高噪声场景下容易过度平滑，细节保留不如DLSS
- 跨平台一致性挑战：不同GPU厂商的驱动实现差异可能导致FSR表现不一致


## 🔮 延伸思考

### 降噪与超分辨率的融合趋势

- 传统管线：光线追踪 → 手动降噪 → 超分辨率（DLSS/FSR），各环节独立优化
- 融合趋势：DLSS 3.5 Ray Reconstruction将降噪和超分辨率合并为一个AI网络，直接从低分辨率光线追踪信号生成高质量图像
- 优势：端到端优化避免了信息在多个阶段之间的损失，AI可以学习到手动设计难以捕捉的跨阶段关联
- 未来方向：可能进一步扩展到整个渲染管线的端到端优化，从G-Buffer直接到最终图像

### 端到端神经渲染的可能性

- 如果AI能够从稀疏采样直接生成高质量图像，传统渲染管线的许多中间步骤可能被简化甚至替代
- 挑战：训练数据的泛化能力、实时推理的计算开销、结果的可控性和确定性
- 潜在方向：Neural Radiance Caching（神经辐射缓存）、Neural Importance Sampling（神经重要性采样）、Neural BRDF Representation
- 业界共识：短期内AI将作为传统管线的增强而非替代，长期来看神经渲染可能重塑整个渲染架构

---

[← 返回 高级渲染技术与前沿方向 题目列表](./index.md) | [返回总纲](../00-总纲.md)
