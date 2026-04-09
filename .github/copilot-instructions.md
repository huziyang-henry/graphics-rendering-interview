# 渲染面试知识体系 — 编码规范

## 数学公式

使用 LaTeX 格式：行内 `$...$`，块级 `$$...$$`（前后空行）。
向量用 `\mathbf{}`，点积用 `\cdot`，函数名用 `\text{}`。
禁止 `\begin{aligned}`、`\begin{pmatrix}`、`\begin{equation}`、`\tag{}`。

## 文件结构

- 面试题 MD 文件必须包含 YAML frontmatter（id, title, chapter, difficulty, knowledge_points, tags）
- 回答结构：结论 → 原理解析 → 工程实践 → 踩坑经验 → 延伸思考
- 文件命名使用英文连字符
- 新增题目后需更新对应章节的 index.md
