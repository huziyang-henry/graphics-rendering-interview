# 04. 材质与纹理系统

> 考察候选人对纹理采样、材质系统设计及GPU纹理架构的理解

---

## 知识点

| 编号 | 知识点 | 关联题目数 |
|------|--------|-----------|
| 04.01 | 纹理采样与过滤（Mipmap/AF） | 3 |
| 04.02 | 纹理压缩格式（BC/ASTC/ETC2） | 1 |
| 04.03 | 纹理寻址模式（Wrap Mode） | 1 |
| 04.04 | PBR材质系统架构 | 1 |
| 04.05 | 虚拟纹理（Virtual Texturing） | 1 |
| 04.06 | 纹理流式加载系统 | 1 |
---

## 题目列表

### 初级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q04.01 | Mipmap的原理是什么？它如何解决纹理缩小时的摩尔纹（Moire）问题？Mipmap会带来多少额外的显存开销？ | 纹理采样与过滤（Mipmap/AF） | [查看](./mipmap-principle-moire-artifact.md) |
| Q04.04 | 纹理环绕模式（Wrap Mode）有哪些？在什么场景下应该使用Clamp而不是Repeat？ | 纹理寻址模式（Wrap Mode） | [查看](./texture-wrap-mode-clamp-repeat.md) |

### 中级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q04.02 | 常见的纹理压缩格式（BC/DXT、ASTC、ETC2）有哪些？它们各自的块大小、压缩比和适用平台是什么？ | 纹理压缩格式（BC/ASTC/ETC2） | [查看](./texture-compression-bc-astc-etc2.md) |
| Q04.03 | 各向异性过滤（Anisotropic Filtering）如何工作？它解决什么问题？开启后对性能有多大影响？ | 纹理采样与过滤（Mipmap/AF） | [查看](./anisotropic-filtering-principle.md) |

### 高级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q04.05 | 请设计一个支持PBR的材质系统架构。如何组织Albedo、Normal、Metallic、Roughness、AO等贴图的绑定和采样？ | PBR材质系统架构 | [查看](./pbr-material-system-architecture.md) |
| Q04.07 | 纹理流式加载（Texture Streaming）系统如何设计？如何根据相机距离和重要性动态调整纹理的Mipmap级别？ | 纹理流式加载系统, 纹理采样与过滤（Mipmap/AF） | [查看](./texture-streaming-system-design.md) |

### 专家级

| 编号 | 题目 | 知识点 | 回答 |
|------|------|--------|------|
| Q04.06 | 虚拟纹理（Virtual Texturing）的原理是什么？它如何解决超大纹理的显存和带宽问题？Page Table和Feedback机制如何工作？ | 虚拟纹理（Virtual Texturing） | [查看](./virtual-texturing-page-table-feedback.md) |

