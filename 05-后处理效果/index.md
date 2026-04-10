# 05. 后处理效果

> 后处理是提升画面品质的关键环节

---

## 知识点

| 编号 | 知识点 | 关联题目数 |
|------|--------|-----------|
| 05.01 | Bloom与Tone Mapping | 2 |
| 05.02 | 屏幕空间技术（SSAO/SSR） | 3 |
| 05.03 | 景深（DOF）与散景模拟 | 1 |
| 05.04 | 时域抗锯齿（TAA） | 1 |
| 05.05 | 后处理管线架构设计 | 1 |
---

## 题目列表

### 中级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q05.01 | Bloom效果的实现原理是什么？请描述从提取亮部到高斯模糊再到最终混合的完整流程。 | Bloom与Tone Mapping | [查看](./bloom-implementation-bright-extract-gaussian-blur.md) |
| Q05.02 | Tone Mapping的作用是什么？ACES曲线和Reinhard曲线各有什么特点？为什么HDR渲染必须做Tone Mapping？ | Bloom与Tone Mapping | [查看](./tone-mapping-aces-reinhard-hdr.md) |
| Q05.03 | SSAO（Screen Space Ambient Occlusion）的原理是什么？它存在哪些视觉缺陷？SSAO的改进版本（如HBAO、GTAO）分别做了哪些优化？ | 屏幕空间技术（SSAO/SSR） | [查看](./ssao-hbao-gtao-screen-space-ao.md) |
| Q05.05 | 景深（Depth of Field，DOF）效果如何实现？请讨论散景（Bokeh）形状的模拟方法。 | 景深（DOF）与散景模拟 | [查看](./depth-of-field-bokeh-simulation.md) |

### 高级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q05.04 | SSR（Screen Space Reflection）的原理是什么？它有哪些局限性（如屏幕外反射丢失）？如何部分弥补这些缺陷？ | 屏幕空间技术（SSAO/SSR） | [查看](./ssr-screen-space-reflection-ray-marching.md) |
| Q05.06 | TAA（Temporal Anti-Aliasing）的原理是什么？它如何利用历史帧信息？运动向量（Motion Vector）在TAA中扮演什么角色？ | 时域抗锯齿（TAA） | [查看](./taa-temporal-anti-aliasing-motion-vector.md) |

### 专家级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q05.07 | 请设计一个完整的后处理管线，考虑Pass的执行顺序、中间RT的精度要求和带宽优化。 | 后处理管线架构设计 | [查看](./post-processing-pipeline-design-pass-order.md) |

