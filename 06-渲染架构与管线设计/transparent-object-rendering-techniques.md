---
id: Q06.10
title: "透明物体是怎么渲染的？"
chapter: 6
chapter_name: "渲染架构与管线设计"
difficulty: intermediate
knowledge_points:
  - id: "06.10"
    name: "透明物体渲染技术全景"
tags: [transparency, alpha-blending, alpha-test, order-independent-transparency, oit]
---

# Q06.10 透明物体是怎么渲染的？

**难度：** 🟡 中级
**所属章节：** [渲染架构与管线设计](./index.md)

---

## 🎯 结论

透明物体渲染的核心难点在于**需要按从后到前的顺序进行混合（Order-Dependent Transparency）**，但 Z-Buffer 无法自然实现这一排序。工业界通过多种方案分层解决：不透明物体用 Alpha Test，简单半透明用深度排序（Sorting），复杂半透明用 Order-Independent Transparency（OIT），特效类用 Additive Blending 或 Dithered Transparency。

---

## 🔍 原理解析

### 为什么透明渲染是个难题

不透明物体使用 Z-Buffer 可以任意顺序渲染——深度测试自动解决遮挡关系。但透明物体需要将片段颜色与背景**混合**（Blend），混合结果取决于绘制顺序：

$$C_{\text{out}} = \alpha_{\text{src}} \cdot C_{\text{src}} + (1 - \alpha_{\text{src}}) \cdot C_{\text{dst}}$$

如果绘制顺序错误，混合结果就不正确。例如：先画近处的红色玻璃（$\alpha = 0.5$），再画远处的蓝色玻璃（$\alpha = 0.5$），与先画蓝色再画红色的结果完全不同。

### 透明渲染方案分类

| 方案 | 原理 | 排序依赖 | 适用场景 | 性能 |
|------|------|---------|---------|------|
| **Alpha Test** | 片段着色器中 `discard` 不满足条件的片段 | 无 | 栅栏、树叶、布料边缘 | 极低 |
| **Alpha Blend + 深度排序** | 按从后到前排序后逐个混合 | 需要排序 | 简单半透明物体（玻璃、水） | 中等 |
| **Additive Blending** | `C_out = C_src + C_dst`（不读取 dst） | 无 | 粒子、光晕、火焰 | 极低 |
| **Dithered Transparency** | 用噪声图案将半透明转为二值 | 无 | 大面积半透明（头发、烟雾） | 低 |
| **OIT（Per-Pixel Linked List）** | 每像素存储链表，最后一次性解析 | 无 | 复杂半透明重叠 | 高 |
| **OIT（Weighted Blended）** | 每像素累加加权颜色和深度权重 | 无 | 通用半透明 | 中等 |
| **Dual Depth Peeling** | 每Pass剥离一层最前面的片段 | 无 | 精确半透明 | 高 |

### Alpha Test vs Alpha Blend

**Alpha Test**（也叫 Alpha Cutoff / Alpha-to-Coverage）：
- 片段要么完全丢弃，要么完全不透明
- 不需要排序，与 Z-Buffer 完美兼容
- 边缘锯齿明显，可用 Alpha-to-Coverage（MSAA）缓解
- 典型应用：树叶、栅栏、布料

**Alpha Blend**：
- 片段与背景按 alpha 值混合
- 需要正确的绘制顺序
- 典型应用：玻璃、水面、烟雾

### OIT 方案详解

**Per-Pixel Linked List（PPLL / A-Buffer）**：
- 每个像素维护一个链表，存储所有通过深度测试的片段
- 所有透明物体渲染完毕后，按深度排序链表中的片段，逐个混合
- 需要额外的 SSBO/Atomic Counter，显存开销大
- 精确但性能开销高

**Weighted Blended OIT**（McGuire 2012）：
- 用两个 Render Target 累加：加权颜色和加权深度
- 权重公式：$w = \alpha \cdot \max(10^{(\text{depth} - \text{near}) \cdot \text{scale}}, 10^{-2})$
- 最后对每个像素做一次除法：$C_{\text{final}} = \frac{C_{\text{accum}}}{w_{\text{accum}}}$
- 近处物体权重更大，远处物体权重更小，近似正确的混合结果
- 不精确但性能好，适合大多数游戏场景

---

## 🛠 工程实践

### 典型游戏引擎的透明渲染策略

```
1. 渲染所有不透明物体（Z-Write ON, Z-Test LESS）
2. 渲染 Alpha Test 物体（Z-Write ON, Alpha Test）
3. 渲染 Additive 粒子（Z-Write OFF, Additive Blend）
4. 渲染半透明物体（Z-Write OFF, Alpha Blend）
   - 简单场景：按物体中心点排序（从后到前）
   - 复杂场景：使用 OIT（Weighted Blended 或 PPLL）
```

### 排序策略

| 策略 | 精度 | 开销 | 适用场景 |
|------|------|------|---------|
| 按物体中心排序 | 低 | 极低 | 简单凸物体（玻璃球） |
| 按物体包围盒最近点排序 | 中 | 低 | 一般半透明物体 |
| 按物体包围盒最远点排序 | 中 | 低 | 大型半透明物体 |
| Per-Triangle 排序 | 高 | 高 | 精确需求（医学可视化） |

### 移动端特殊处理

- 移动端 TBDR 架构下，Alpha Blend 会打断 Tile 的 Early-Z 优化
- 尽量用 Alpha Test + Alpha-to-Coverage 替代 Alpha Blend
- 粒子效果优先使用 Additive Blending（不需要排序）
- 头发/草等大面积半透明使用 Dithered Transparency

---

## ⚠️ 踩坑经验

### 排序不完美的视觉瑕疵

- 按物体中心排序时，两个交叉的半透明物体无论哪种顺序都不完全正确
- 解决方案：对交叉物体使用 OIT，或美术上避免设计交叉半透明物体

### Z-Fighting 问题

- 半透明物体关闭 Z-Write 后，多个半透明物体在同一深度会出现 Z-Fighting
- 解决方案：对半透明物体添加微小的深度偏移（Polygon Offset）

### 粒子排序问题

- 大量粒子按中心点排序开销高且效果差
- 解决方案：粒子使用 Additive Blending（无需排序），或按距离分桶粗排

### 后处理与透明物体的交互

- 后处理（如 Bloom、DOF）在透明物体之前应用，会导致透明物体不受后处理影响
- 解决方案：将需要后处理的半透明物体单独分层，延迟到后处理之后混合

---

## 💡 延伸思考

### 与延迟渲染的关系

延迟渲染天然不适合处理半透明物体（因为 G-Buffer 只存储一个不透明片段）。解决方案包括：
- 在延迟 Pass 之后单独用前向渲染绘制半透明物体
- 使用 Depth Peeling 等多层 G-Buffer 方案
- 使用 Forward+ 管线统一处理

### 前沿方向

- **ReSTIR**：基于光线的重采样技术，可以更高效地处理参与多次散射的半透明介质
- **Transmittance Estimation**：实时估算参与介质（云、雾、皮肤）的光透射
- **Virtual Shadow Maps**：UE5 中半透明物体的阴影也可以通过虚拟阴影贴图获得精确阴影
