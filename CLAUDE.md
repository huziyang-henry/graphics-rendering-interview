# 渲染面试知识体系 — 项目规范

本文件为本仓库的 LLM 编码规范，Claude Code、Trae、Cursor 等工具会自动加载。

## 项目概述

这是一个面向游戏引擎工程师的图形渲染面试知识库，采用 Markdown + YAML frontmatter 结构，托管在 GitHub 上。

## 文件结构

```
渲染面试知识体系/
├── 00-总纲.md              # 全局导航
├── XX-章节名/index.md      # 章节索引（知识点表 + 题目索引）
├── XX-章节名/*.md          # 单题回答（含 YAML frontmatter）
├── CLAUDE.md               # 本文件（LLM 规范）
└── .cursor/rules/          # Cursor 专用规则
```

## Markdown 编写规范

### 数学公式（LaTeX）

本仓库所有 Markdown 文件的数学公式使用标准 LaTeX 格式，GitHub 通过 MathJax 渲染。

**行内公式**：`$...$`，用于简单表达式
**块级公式**：`$$...$$`（前后各留一个空行），用于含分数/积分的完整公式

**符号约定**：
- 向量：`\mathbf{n}`、`\mathbf{l}`、`\mathbf{v}`、`\mathbf{h}`
- 点积：`\mathbf{n} \cdot \mathbf{l}`
- 函数名：`\text{roughness}`、`\text{albedo}`、`\text{normalize}`
- 希腊字母：`\alpha`、`\theta`、`\pi`、`\lambda`、`\sigma`

**GitHub 不兼容的 LaTeX 环境（禁止使用）**：
- `\begin{aligned}` → 用 `\\` 换行
- `\begin{pmatrix}` → 用 `\begin{bmatrix}`
- `\begin{equation}` → 用 `$$...$$`
- `\tag{}` → 用文字标注

**格式要求**：
- 行内公式 `$` 前后各留一个半角空格
- 块级公式前后各留一个空行

### YAML Frontmatter

每道面试题的 MD 文件头部必须包含 YAML frontmatter：

```yaml
---
id: Q03.01
title: "题目名称"
chapter: 3
chapter_name: "章节名称"
difficulty: beginner          # beginner | intermediate | advanced | expert
knowledge_points:
  - id: "03.01"
    name: "知识点名称"
tags: [tag1, tag2]
---
```

### 五步回答结构

每道题的回答遵循：结论 → 原理解析 → 工程实践 → 踩坑经验 → 延伸思考

### 文件命名

- 面试题文件使用英文连字符命名：`shadow-map-principle-shadow-acne-peter-panning.md`
- 章节目录使用中文编号前缀：`03-阴影技术/`

## 新增内容检查清单

- [ ] YAML frontmatter 格式正确且字段完整
- [ ] 数学公式使用 `$` 或 `$$` 包裹
- [ ] 向量统一使用 `\mathbf{}`
- [ ] 行内公式前后有空格，块级公式前后有空行
- [ ] 文件命名使用英文连字符
- [ ] 更新对应章节的 index.md
