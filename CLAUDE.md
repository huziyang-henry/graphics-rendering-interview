# 渲染面试知识体系 — 项目规范

> 本文件是索引，详细规范见 `.claude/docs/` 下的子文件。
> Claude Code 会自动加载 `.claude/docs/` 下所有 `.md` 文件。
> **修改规范时，必须同步更新三个位置的文件**（见下方列表）。

## 项目概述

面向游戏引擎工程师的图形渲染面试知识库，Markdown + YAML frontmatter 结构，托管在 GitHub。

## 规范文件清单

| 文件 | 内容 |
|------|------|
| `.claude/docs/workflow.md` | 面试题生成工作流、新增题目工作流、编号规则 |
| `.claude/docs/frontmatter.md` | YAML Frontmatter 字段规范、知识点关系 |
| `.claude/docs/answer-structure.md` | 五步回答结构（结论→原理→工程→踩坑→延伸） |
| `.claude/docs/latex-math.md` | LaTeX 符号约定、行内/块级判定、GitHub 兼容性、单字母陷阱 |
| `.claude/docs/file-naming.md` | 文件命名规范 |
| `.claude/docs/quality.md` | 内容质量要求、截断行检查、Unicode 转义检查 |
| `.claude/docs/git-conventions.md` | Git 提交规范、分支/PR 工作流 |
| `.claude/docs/retrospective.md` | 经验沉淀规范、同步策略 |
| `.claude/docs/checklist.md` | 新增内容检查清单 |

## 同步更新要求

修改规范时，必须同步更新以下三个位置：

1. `.claude/docs/` 下对应的子文件（**主源**）
2. `.cursor/rules/project-rules.md`（Cursor 合并完整版）
3. `.github/copilot-instructions.md`（Copilot/Codex 合并完整版）

**同步方向**：先改 `.claude/docs/` 子文件，再合并到两个完整版文件。
