# 渲染面试知识体系 — 编码规范

## 新增题目流程

1. 在对应章节目录创建 `.md` 文件（英文连字符命名）
2. 编写 YAML frontmatter（id, title, chapter, chapter_name, difficulty, knowledge_points, tags）
3. 按五步结构撰写：结论 → 原理解析 → 工程实践 → 踩坑经验 → 延伸思考
4. 更新对应章节的 `index.md`
5. 不要修改 `00-总纲.md`

## 数学公式

- LaTeX 格式：行内 `$...$`，块级 `$$...$$`（前后空行）
- 向量 `\mathbf{}`，点积 `\cdot`，函数名 `\text{}`
- 禁止 `\begin{aligned}`、`\begin{pmatrix}`、`\begin{equation}`、`\tag{}`
- 单字母变量（如 `$n$`）在 GitHub 不渲染，用 Markdown 斜体 `*n*` 代替

## YAML Frontmatter

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

## 质量要求

- 每步回答至少 3-5 行实质内容
- GLSL/HLSL 代码不转 LaTeX
- 文件名禁止中文括号、空格、全角标点
- 无截断行（`\` 结尾）和 Unicode 转义残留
