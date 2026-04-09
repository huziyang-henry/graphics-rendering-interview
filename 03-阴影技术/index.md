# 03. 阴影技术

> 阴影是实时渲染中最重要的视觉元素之一

---

## 知识点

| 编号 | 知识点 | 关联题目数 |
|------|--------|-----------|
| 03.01 | Shadow Map基础原理 | 2 |
| 03.02 | 软阴影技术（PCF/PCSS/VSM/ESM） | 3 |
| 03.03 | 级联阴影贴图（CSM） | 2 |
| 03.04 | 虚拟阴影贴图（Virtual Shadow Maps） | 1 |
| 03.05 | 开放世界阴影系统设计 | 1 |
---

## 题目列表

### 初级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q03.01 | Shadow Map的基本原理是什么？它存在哪些经典问题（如阴影粉刺Shadow Acne、彼得潘现象Peter Panning）？如何解决？ | Shadow Map基础原理 | [查看](./shadow-map-principle-shadow-acne-peter-panning.md) |

### 中级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q03.02 | 什么是PCF（Percentage Closer Filtering）？PCSS（Percentage Closer Soft Shadows）相比PCF有什么改进？ | 软阴影技术（PCF/PCSS/VSM/ESM） | [查看](./pcf-pcss-soft-shadow.md) |
| Q03.03 | 级联阴影贴图（CSM）的原理是什么？如何确定级联的划分策略（对数划分 vs 均匀划分 vs PSSM）？ | 级联阴影贴图（CSM） | [查看](./csm-cascade-shadow-maps-split-scheme.md) |
| Q03.04 | Shadow Map的分辨率不足时会出现什么问题？除了提高分辨率，还有哪些方法可以改善阴影质量？ | Shadow Map基础原理, 软阴影技术（PCF/PCSS/VSM/ESM） | [查看](./shadow-map-resolution-optimization.md) |

### 高级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q03.05 | VSM（Variance Shadow Map）和ESM（Exponential Shadow Map）的原理分别是什么？它们各自有什么优缺点？ | 软阴影技术（PCF/PCSS/VSM/ESM） | [查看](./vsm-esm-filterable-shadow.md) |
| Q03.06 | 在大型开放世界中，如何设计一个高效的阴影系统？请讨论CSM与距离场阴影的配合策略。 | 级联阴影贴图（CSM）, 开放世界阴影系统设计 | [查看](./open-world-shadow-system-design.md) |

### 专家级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q03.07 | 虚拟阴影贴图（Virtual Shadow Maps，如UE5的实现）的核心思想是什么？它如何解决传统CSM的分辨率分配不均问题？ | 虚拟阴影贴图（Virtual Shadow Maps） | [查看](./virtual-shadow-maps-ue5.md) |

