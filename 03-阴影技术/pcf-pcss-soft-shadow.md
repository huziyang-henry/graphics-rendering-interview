---
id: Q03.02
title: "什么是PCF（Percentage Closer Filtering）？PCSS（Percentage Closer Soft Shadows）相比PCF有什么改进？"
chapter: 3
chapter_name: "阴影技术"
difficulty: intermediate
knowledge_points:
  - id: "03.02"
    name: "软阴影技术（PCF/PCSS/VSM/ESM）"
tags: ["pcf", "pcss", "soft-shadow", "poisson-disk"]
---

# Q03.02 什么是PCF（Percentage Closer Filtering）？PCSS（Percentage Closer Soft Shadows）相比PCF有什么改进？

**难度：** 🟡 中级
**所属章节：** [阴影技术](./index.md)

---

## 🎯 结论

- PCF（Percentage Closer Filtering）是实时渲染中实现软阴影边缘的基础技术，通过对Shadow Map邻域进行多次深度比较并取平均值，使阴影边缘产生柔和的半影（Penumbra）过渡效果。
- PCSS（Percentage Closer Soft Shadows）在PCF的基础上引入了物理正确的半影宽度计算，根据遮挡物与接收面的距离动态调整采样半径，从而生成物理上更准确的软阴影。

## 📐 原理解析

### PCF的工作原理

- 标准Shadow Map的深度比较是二值化的（0或1），导致阴影边缘呈现锯齿状的硬边界。PCF的核心思想是对Shadow Map中当前像素周围的N个采样点分别进行深度比较，得到N个0/1结果，然后取平均值作为最终的阴影因子（Shadow Factor）。
- 例如，对一个3x3的采样核进行PCF滤波，如果9个采样点中有5个被判定为阴影，则阴影因子为5/9 ≈ 0.56，该像素呈现半阴影状态。采样核越大，阴影边缘越柔和，但计算开销也越大。
- PCF本质上是一个对深度测试结果的均值滤波（Box Filter），它不等同于对深度值本身做滤波——先滤波深度再比较会产生错误的结果（因为深度比较是非线性操作）。

### PCSS的三阶段流程

- 第一阶段——Blocker Search（遮挡物搜索）：在Shadow Map中以当前像素为中心，在给定搜索半径内采样，找出所有深度值小于接收面深度的采样点（即遮挡物），计算它们的平均深度z_blocker。如果找不到遮挡物，则该像素完全处于光照中，无需后续处理。
- 第二阶段——Penumbra Estimation（半影宽度估算）：根据相似三角形原理，半影宽度w_penumbra = (z_receiver - z_blocker) * w_light / z_blocker，其中w_light是光源的表观宽度。遮挡物越远离接收面，半影越宽。
- 第三阶段——PCF Filtering：使用估算出的半影宽度作为PCF的采样半径，在Shadow Map中进行PCF滤波，生成软阴影。半影宽的区域使用大半径采样，半影窄的区域使用小半径采样。


## 🛠 工程实践

### PCF的采样模式优化

- 规则网格采样（Regular Grid）：最简单的采样模式，但容易产生方向性的条带伪影。
- Poisson Disk采样：在单位圆内预计算一组满足泊松分布的采样点，具有更好的各向同性，能有效减少条带伪影。通常使用16-64个采样点。
- Vogel Disk采样：基于黄金角度的螺旋采样模式，在采样点数量变化时能保持较好的分布质量，适合需要动态调整采样数量的场景。
- 硬件PCF：DirectX 9+和OpenGL支持通过比较滤波器（Comparison Filter）在硬件层面实现2x2的PCF，性能开销极低，但采样核大小受限。

### PCSS的参数调优

- Blocker Search的搜索半径应设置为可能的最大半影宽度，通常与光源大小和场景尺度相关。搜索步长可以采用逐级放大（逐步增加搜索范围）的策略来减少采样次数。
- PCF阶段的最大采样半径需要设置上限，避免在遮挡物非常远时采样半径过大导致性能问题。通常限制在32-64个采样点以内。
- 可以使用Early Bailout优化：如果在Blocker Search阶段发现没有遮挡物，直接跳过后续阶段。


## ⚠️ 踩坑经验

### PCF采样数不足的条带伪影

当PCF采样点数量较少（如少于16个）时，阴影边缘会出现明显的规律性条纹，尤其在采样模式不够随机时更为严重。实践中建议至少使用16个Poisson Disk采样点，并配合旋转随机化（Rotation Randomization）来打破固定模式。在移动端受限于性能预算，可以使用4-8个采样点配合Tent Filter（三角滤波器）来改善效果。

### PCSS的Blocker Search误判

- 在复杂场景中，Blocker Search可能将多个不连续的遮挡物误判为一个整体，导致半影宽度估算不准确。例如，当遮挡物和接收面之间存在间隙时，搜索到的遮挡物深度可能偏大，导致半影宽度被高估。
- Blocker Search本身也需要较多的采样次数（通常16-32次），在性能敏感的场景中可能成为瓶颈。可以使用交错搜索（Interleaved Search）或预计算搜索模式来优化。
- 对于面积光源，PCSS假设光源是均匀发光的圆盘，对于非均匀或复杂形状的光源，半影估算的物理准确性会下降。


## 🔮 延伸思考

### Variance Soft Shadow Maps（VSSM）

VSSM利用VSM（Variance Shadow Map）的均值和方差信息来估算半影宽度，避免了PCSS中昂贵的Blocker Search。通过单次Mipmap查找即可获得一个区域内的遮挡物分布信息，大大降低了采样次数。但VSSM继承了VSM的光晕（Light Bleeding）问题，需要额外的矫正手段。

### Screen Space Shadows的思路

Screen Space Shadows（如Unity的Screen Space Shadows）在屏幕空间而非光源空间进行阴影计算，避免了Shadow Map的分辨率分配问题。通过屏幕空间的深度缓冲进行光线步进（Ray Marching）来检测遮挡。这种方法不受Shadow Map分辨率的限制，但受限于屏幕空间信息的完整性——不在屏幕中的物体无法投射阴影，且屏幕空间深度缓冲的精度有限。

---

[← 返回 阴影技术 题目列表](./index.md) | [返回总纲](../00-总纲.md)
