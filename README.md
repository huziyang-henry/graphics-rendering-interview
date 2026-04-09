# 渲染面试知识体系 (Graphics Rendering Interview Knowledge Base)

> 面向游戏引擎工程师的图形渲染面试题库，采用 LLM-friendly 结构化设计，方便人维护，也方便大模型处理。

## 结构说明

```
渲染面试知识体系/
├── 00-总纲.md                    # 全局导航：章节索引 + 难度图例 + 维护指南
├── 01-渲染管线基础/               # Chapter 1: Rendering Pipeline
│   ├── index.md                  # 章节索引：知识点表 + 按难度分组的面试题
│   └── *.md                      # 各面试题详细回答
├── 02-光照与着色模型/             # Chapter 2: Lighting & Shading
├── 03-阴影技术/                   # Chapter 3: Shadow Techniques
├── 04-材质与纹理系统/             # Chapter 4: Materials & Textures
├── 05-后处理效果/                 # Chapter 5: Post-Processing
├── 06-渲染架构与管线设计/         # Chapter 6: Rendering Architecture
├── 07-性能优化与GPU架构/          # Chapter 7: GPU Optimization
├── 08-高级渲染技术与前沿方向/     # Chapter 8: Advanced Techniques
└── 09-综合场景与系统设计/         # Chapter 9: System Design
```

## 三层导航体系

| 层级 | 文件 | 职责 |
|------|------|------|
| L1 总纲 | `00-总纲.md` | 全局导航入口，章节概览 |
| L2 章节索引 | `XX-章节名/index.md` | 知识点表 + 面试题索引 |
| L3 题目详情 | `XX-章节名/*.md` | 单题完整回答 |

## 回答结构

每道面试题遵循统一的五步回答结构：

1. **结论** — 一句话核心答案
2. **原理解析** — 技术原理深入分析
3. **工程实践** — 实际项目中的落地方式
4. **踩坑经验** — 常见问题与解决方案
5. **延伸思考** — 进阶方向与关联技术

## YAML Frontmatter 规范

每道题目的 Markdown 文件头部包含结构化元数据：

```yaml
---
id: Q03.01
title: "Shadow Map的基本原理是什么？"
chapter: 3
chapter_name: "阴影技术"
difficulty: beginner          # beginner | intermediate | advanced | expert
knowledge_points:
  - id: "03.01"
    name: "Shadow Map 基础原理"
tags: [shadow, shadow-map]
---
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 题目唯一标识，格式 `Q{章节}.{序号}` |
| `title` | string | 题目（中文） |
| `chapter` | number | 章节编号 |
| `chapter_name` | string | 章节名称 |
| `difficulty` | enum | 难度等级：beginner / intermediate / advanced / expert |
| `knowledge_points` | array | 关联的知识点列表（多对多关系） |
| `tags` | array | 英文标签，便于搜索和分类 |

## 维护指南

### 新增面试题

1. 在对应章节目录下创建新的 `.md` 文件（英文连字符命名）
2. 添加 YAML frontmatter 元数据
3. 按五步结构撰写回答
4. 更新对应章节的 `index.md`，在题目索引表中添加条目

### 新增知识点

1. 在对应章节的 `index.md` 知识点表中添加条目
2. 在关联的面试题 frontmatter 中添加 `knowledge_points` 引用

### LLM 协作建议

- 本仓库的 YAML frontmatter 为 LLM 提供结构化上下文，可直接用于 RAG 检索
- 修改内容时请保持 frontmatter 与正文的一致性
- 新增文件请遵循现有的命名和格式规范

## 统计

- **9 个章节**，覆盖渲染管线、光照、阴影、材质、后处理、架构、优化、前沿技术、系统设计
- **76 道面试题**，难度分布从 beginner 到 expert
- **60+ 个知识点**，与面试题多对多关联

## License

Private repository. All rights reserved.
