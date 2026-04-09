---
id: Q04.02
title: "常见的纹理压缩格式（BC/DXT、ASTC、ETC2）有哪些？它们各自的块大小、压缩比和适用平台是什么？"
chapter: 4
chapter_name: "材质与纹理系统"
difficulty: intermediate
knowledge_points:
  - id: "04.02"
    name: "纹理压缩格式（BC/ASTC/ETC2）"
tags: ["bc", "dxt", "astc", "etc2", "texture-compression"]
---

# Q04.02 常见的纹理压缩格式（BC/DXT、ASTC、ETC2）有哪些？它们各自的块大小、压缩比和适用平台是什么？

**难度：** 🟡 中级
**所属章节：** [材质与纹理系统](./index.md)

---

## 🎯 结论

- 纹理压缩是实时渲染中减少显存占用和带宽消耗的关键技术。三大主流压缩格式为：BC系列（Block Compression，PC/Xbox平台标准）、ASTC（Adaptive Scalable Texture Compression，移动端通用标准）、ETC2（Ericsson Texture Compression 2，移动端OpenGL ES 3.0标准格式）。
- 所有这些格式都基于块压缩（Block Compression）原理，将纹理划分为固定大小的块，每块独立编解码，支持GPU硬件随机访问。

## 📐 原理解析

### BC系列（Block Compression / DXT）

- BC1（DXT1）：4x4像素块，每块64bit（2个16色端点 + 16个2bit索引），压缩比4:1。仅支持RGB，不支持Alpha或仅支持1bit透明。适用于Albedo/Diffuse贴图。
- BC2（DXT3）：4x4像素块，每块128bit。RGB部分同BC1，额外64bit存储4x4的4bit独立Alpha值。适用于需要锐利Alpha边缘的贴图。
- BC3（DXT5）：4x4像素块，每块128bit。RGB部分同BC1，额外64bit存储2个Alpha端点 + 16个3bit索引的插值Alpha。Alpha质量优于BC2，适用于渐变Alpha贴图。
- BC4：4x4像素块，每块64bit。单通道（R），存储1个8bit端点和16个3bit索引。适用于灰度贴图、高度图。
- BC5（3DC/ATI2）：4x4像素块，每块128bit。双通道（RG），由两个BC4块组成。适用于法线贴图（存储XY法线，Z从XY重建）。
- BC6H：4x4像素块，每块128bit。支持HDR浮点纹理（16bit half-float RGB），适用于HDR环境贴图、天空盒。
- BC7：4x4像素块，每块128bit。支持RGBA，提供多种模式（固定bitrate 8bpp），质量远高于BC1-BC3。适用于高质量Albedo和需要Alpha的高质量贴图。

### ASTC（Adaptive Scalable Texture Compression）

- 由ARM和AMD联合开发，支持可变块大小，从4x4到12x12（甚至3x3），对应压缩比从3.56:1到12:1。
- 常用配置：ASTC 4x4（8bpp，质量最高）、ASTC 6x6（约3.56bpp）、ASTC 8x8（2bpp）、ASTC 10x10（1.28bpp）、ASTC 12x12（约0.89bpp）。
- 支持LDR（Low Dynamic Range）和HDR模式，支持1-4通道（包括3D纹理和2D Array）。
- 使用Bisection算法进行颜色空间分区，可以在一个块内将像素分为不同区域分别编码，因此在低bitrate下仍能保持较好的边缘质量。
- 适用平台：移动端（ARM Mali GPU、Qualcomm Adreno 5xx+、Apple GPU）、Nintendo Switch、部分桌面GPU（Intel Arc、AMD RDNA）。

### ETC2（Ericsson Texture Compression 2）

- ETC2是ETC1的超集，作为OpenGL ES 3.0和WebGL 2.0的强制标准格式。
- ETC2 RGB：4x4像素块，每块64bit，压缩比4:1。相比ETC1增加了两种新模式（T/H模式）用于处理高对比度边缘。
- ETC2 EAC R11/RG11：单通道/双通道，每块64bit/128bit。适用于法线贴图（RG两个通道）和高度图（R单通道）。
- ETC2 EAC RGBA8：4x4像素块，每块128bit，RGB部分使用ETC2编码，Alpha部分使用EAC编码。
- 适用平台：移动端（OpenGL ES 3.0+设备）、WebGL 2.0。注意：iOS设备虽然支持ETC2，但更推荐使用ASTC以获得更好的质量和灵活性。


## 🛠 工程实践

### 跨平台纹理格式选择策略

- PC/Xbox平台：优先使用BC7（高质量RGBA）和BC5（法线贴图），BC6H用于HDR贴图。BC1用于不需要Alpha的Albedo贴图。
- Android平台：优先使用ASTC（如果设备支持），否则回退到ETC2。需要做运行时格式检测（通过GPU能力查询）。
- iOS平台：优先使用ASTC（所有Apple GPU均支持），其次ETC2。
- Nintendo Switch：使用ASTC或BC（Switch的NVIDIA GPU两者都支持）。
- Web平台：WebGL 2.0保证支持ETC2（通过EXT_texture_compression_etc2扩展），ASTC需要额外扩展支持。

### 压缩工具链

- Compressonator（AMD开源）：支持BC、ASTC、ETC2全系列格式，提供命令行和GUI工具，适合集成到构建管线。
- ASTC Encoder（ARM开源）：ASTC格式的参考编码器，支持质量-性能权衡的编码预设（-thoroughness参数）。
- texconv（Microsoft DirectXTex）：Windows平台下BC系列格式的高质量编码工具，广泛用于DirectX项目。
- NVIDIA Texture Tools Exporter：支持CUDA加速的BC7编码，质量高且速度快。
- PVRTexTool（Imagination）：支持PVRTC、ASTC、ETC2，适合移动端开发。

### 质量与压缩率的权衡

对于Albedo贴图，通常使用最高质量设置（如BC7或ASTC 4x4），因为颜色细节直接影响视觉质量。对于法线贴图，使用BC5或ASTC 4x4/5x4，确保法线精度。对于粗糙度/金属度等单通道贴图，可以使用较低质量（ASTC 6x6或8x8），因为这些贴图变化平缓，压缩伪影不明显。环境贴图（Cubemap）使用BC6H或ASTC HDR以保留高动态范围。


## ⚠️ 踩坑经验

### 块状伪影（Block Artifact）问题

所有基于块的压缩格式在高对比度边缘（如文字、UI元素、细线条）处都会出现明显的块状伪影。这是因为每个4x4块只能表示有限的颜色范围。解决方案：对UI纹理使用不压缩格式（RGBA8）或使用专门的压缩工具进行质量优化；在美术工作流中避免在高对比度贴图上使用过度压缩。

### ASTC在低bitrate下的质量下降

当使用ASTC 8x8或更低质量时，法线贴图和法线相关的细节会明显退化。特别是在低频渐变区域，可能出现色带（Banding）现象。建议法线贴图至少使用ASTC 5x4或4x4，粗糙度/金属度可以使用ASTC 6x6或8x8。

### 法线贴图的专用压缩格式

法线贴图不能使用标准的RGB压缩格式（如BC1/ETC2 RGB），因为块压缩的颜色插值会导致法线方向偏移，产生光照错误。必须使用双通道格式（BC5/ETC2 RG/ASTC 2D），在着色器中从XY重建Z分量（Z = sqrt(1 - X*X - Y*Y)），确保法线的单位长度约束。

### Alpha通道处理陷阱

BC1的1bit Alpha在渐变透明区域会产生锯齿状硬边。需要平滑Alpha时必须使用BC3（DXT5）或BC7。此外，BC3的Alpha插值在某些情况下可能导致"预乘Alpha"问题，需要注意着色器中的Alpha混合模式是否与压缩格式匹配。


## 🔮 延伸思考

### GPU纹理压缩的未来方向

传统的块压缩格式设计于固定功能管线时代，难以适应现代渲染中多样化的数据类型需求。未来的方向包括：自适应压缩率（根据纹理内容自动调整不同区域的压缩质量）、基于机器学习的压缩（利用神经网络学习更优的块编码策略）、以及针对特定数据类型（如Spherical Harmonics、Distance Field）的专用压缩格式。

### 纹理压缩对带宽的实际影响

纹理采样是GPU内存带宽的主要消耗者。使用压缩格式不仅减少显存占用，更重要的是减少实际读取的数据量——GPU只解码被采样到的texel所在块，未采样的块不会被读取。在4K分辨率下，不压缩的纹理采样可能消耗数十GB/s的带宽，而压缩后可降低到原来的1/4到1/8。这对移动端GPU尤其关键，因为移动设备的内存带宽远低于桌面端。

### 纹理压缩与质量验证管线

在实际项目中，建议建立自动化的压缩质量验证管线：对每张贴图在压缩前后计算PSNR/SSIM指标，当质量低于阈值时自动报警。同时，在美术Review流程中提供压缩预览工具，让美术人员能在DCC工具中直接看到压缩后的效果，避免后期才发现压缩质量问题。

---

[← 返回 材质与纹理系统 题目列表](./index.md) | [返回总纲](../00-总纲.md)
