# 渲染面试知识体系 — 项目规范

> 本文件是索引，详细规范见 `.claude/docs/` 下的子文件。
> Claude Code 会自动加载 `.claude/docs/` 下所有 `.md` 文件。
> **修改规范时，必须同步更新三个位置的文件**（见下方列表）。

## 项目概述

面向游戏引擎工程师的图形渲染面试知识库，Markdown + YAML frontmatter 结构，托管在 GitHub。

## 规范文件清单（三端分文件，内容一致）

| 子文件 | Claude Code | Cursor | Copilot/Codex | 内容 |
|--------|------------|--------|---------------|------|
| `workflow` | `.claude/docs/workflow.md` | `.cursor/rules/workflow.mdc` | `.github/instructions/workflow.instructions.md` | 面试题生成工作流、新增题目工作流、编号规则 |
| `frontmatter` | `.claude/docs/frontmatter.md` | `.cursor/rules/frontmatter.mdc` | `.github/instructions/frontmatter.instructions.md` | YAML Frontmatter 字段规范、知识点关系 |
| `answer-structure` | `.claude/docs/answer-structure.md` | `.cursor/rules/answer-structure.mdc` | `.github/instructions/answer-structure.instructions.md` | 五步回答结构 |
| `latex-math` | `.claude/docs/latex-math.md` | `.cursor/rules/latex-math.mdc` | `.github/instructions/latex-math.instructions.md` | LaTeX 符号约定、GitHub 兼容性 |
| `file-naming` | `.claude/docs/file-naming.md` | `.cursor/rules/file-naming.mdc` | `.github/instructions/file-naming.instructions.md` | 文件命名规范 |
| `quality` | `.claude/docs/quality.md` | `.cursor/rules/quality.mdc` | `.github/instructions/quality.instructions.md` | 内容质量要求 |
| `git-conventions` | `.claude/docs/git-conventions.md` | `.cursor/rules/git-conventions.mdc` | `.github/instructions/git-conventions.instructions.md` | Git 提交规范、分支/PR 工作流 |
| `retrospective` | `.claude/docs/retrospective.md` | `.cursor/rules/retrospective.mdc` | `.github/instructions/retrospective.instructions.md` | 经验沉淀规范、同步策略 |
| `checklist` | `.claude/docs/checklist.md` | `.cursor/rules/checklist.mdc` | `.github/instructions/checklist.instructions.md` | 新增内容检查清单 |

## 同步更新要求

修改规范时，必须同步更新以下三个位置：

1. `.claude/docs/` 下对应的子文件（**主源**）
2. `.cursor/rules/` 下对应的 `.mdc` 文件
3. `.github/instructions/` 下对应的 `.instructions.md` 文件

**同步方向**：先改 `.claude/docs/` 子文件（主源），再同步到 Cursor 和 Copilot 对应文件。
**内容一致**：三个文件正文内容完全一致，仅 frontmatter 格式不同。
