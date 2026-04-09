# 渲染面试知识体系 — 项目规范

> 本文件是本仓库的 LLM 编码规范。Claude Code、Trae 等工具会自动加载此文件。
> Cursor 用户请参阅 `.cursor/rules/latex-math-rules.md`。
> GitHub Copilot 用户请参阅 `.github/copilot-instructions.md`。

---

## 项目概述

面向游戏引擎工程师的图形渲染面试知识库，Markdown + YAML frontmatter 结构，托管在 GitHub。

```
渲染面试知识体系/
├── 00-总纲.md              # 全局导航（纯链接，不维护数字）
├── XX-章节名/index.md      # 章节索引（知识点表 + 按难度分组的题目索引）
├── XX-章节名/*.md          # 单题回答（YAML frontmatter + 五步结构正文）
├── CLAUDE.md               # 本文件
├── .cursor/rules/          # Cursor 专用规则
└── .github/copilot-instructions.md  # Copilot/Codex 专用规则
```

---

## 一、新增题目工作流

1. 确定题目归属章节和难度等级
2. 在对应章节目录下创建 `.md` 文件（英文连字符命名）
3. 编写 YAML frontmatter + 五步结构正文
4. 在对应章节的 `index.md` 中添加题目索引行和知识点条目（如有新知识点）
5. **不要**修改 `00-总纲.md`（总纲不维护数字，只保留导航链接）

### 编号规则

- 格式：`Q{章节号}.{序号}`，如 `Q03.05`
- 章节号两位数补零，序号不补零
- 编号一旦分配不再变更

---

## 二、YAML Frontmatter 规范

每道面试题必须包含完整的 YAML frontmatter：

```yaml
---
id: Q03.01
title: "题目名称（中文）"
chapter: 3
chapter_name: "阴影技术"
difficulty: beginner
knowledge_points:
  - id: "03.01"
    name: "知识点名称"
tags: [shadow, shadow-map]
---
```

### 字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| `id` | 是 | 唯一标识，`Q{章节}.{序号}` |
| `title` | 是 | 中文题目，用双引号包裹 |
| `chapter` | 是 | 章节编号（数字） |
| `chapter_name` | 是 | 章节名称（中文） |
| `difficulty` | 是 | 枚举值：`beginner` / `intermediate` / `advanced` / `expert` |
| `knowledge_points` | 是 | 关联知识点列表（多对多关系） |
| `tags` | 是 | 英文标签数组，用于搜索和分类 |

### 知识点与题目的关系

- 多对多关系：一个知识点可对应多道题，一道题可关联多个知识点
- 知识点不绑定难度等级（同一知识点可以有初级和高级题目）
- 知识点 ID 格式：`{章节号}.{序号}`，如 `03.01`

---

## 三、五步回答结构

每道题的回答使用 `###` 标题，遵循以下结构：

1. **结论** — 一句话核心答案，开门见山
2. **原理解析** — 技术原理深入分析，可含数学推导
3. **工程实践** — 实际项目中的选型、实现和优化
4. **踩坑经验** — 真实项目中遇到的问题和解决方案
5. **延伸思考** — 对比同类技术、前沿方向、关联问题

每步至少 3-5 行详细说明，避免空洞。

---

## 四、数学公式规范（LaTeX）

GitHub 通过 MathJax 渲染，使用 `$...$` 行内和 `$$...$$` 块级。

### 符号约定

| 类型 | LaTeX | 示例 |
|------|-------|------|
| 向量 | `\mathbf{}` | `\mathbf{n}`, `\mathbf{l}`, `\mathbf{v}`, `\mathbf{h}` |
| 点积 | `\cdot` | `\mathbf{n} \cdot \mathbf{l}` |
| 分数 | `\frac{}{}` | `\frac{\text{albedo}}{\pi}` |
| 积分 | `\int_{\Omega}` | 渲染方程 |
| 函数名 | `\text{}` | `\text{roughness}`, `\text{normalize}` |
| 希腊字母 | LaTeX 命令 | `\alpha`, `\theta`, `\pi`, `\lambda` |

### 行内 vs 块级

- **行内 `$...$`**：简单代数、向量运算、复杂度、参数引用
- **块级 `$$...$$`**：含分数、积分、矩阵的完整公式；前后各留一个空行

### ⚠️ GitHub 兼容性注意事项

**禁止使用**（GitHub MathJax 不支持）：
- `\begin{aligned}` → 用 `\\` 换行
- `\begin{pmatrix}` → 用 `\begin{bmatrix}`
- `\begin{equation}` → 用 `$$...$$`
- `\tag{}` → 用文字标注
- `\label{}` / `\ref{}` → 用文字引用

**单字母变量陷阱**：
- `$n$`、`$f$`、`$k$` 等仅含单个普通字母的行内公式在 GitHub 上**不会被渲染**
- 解决方案：使用 Markdown 斜体 `*n*` 代替 `$n$`，或将其放入更完整的公式中

**格式要求**：
- 行内公式 `$` 前后各留一个半角空格（中文与 `$` 之间也留空格）
- 块级公式前后各留一个空行

---

## 五、文件命名规范

- 面试题文件：英文连字符命名，如 `shadow-map-principle-shadow-acne-peter-panning.md`
- 章节目录：中文编号前缀，如 `03-阴影技术/`
- 禁止在文件名中使用中文括号 `（）`、空格、全角标点

---

## 六、内容质量要求

- **不生成空洞内容**：每步回答至少 3-5 行实质内容
- **代码与公式分离**：GLSL/HLSL 代码片段不转为 LaTeX，保持代码块格式
- **截断行检查**：以 `\` 结尾的行视为截断，需补全
- **Unicode 转义检查**：`u201C`、`u201D` 等应转为实际字符 `"` `"`
- **中文排版**：中文与英文/数字之间建议加空格（不加也不算错）

---

## 七、Git 提交规范

- 按功能分批提交，便于审查和回溯
- Commit message 使用英文，格式：`{type}: {简短描述}`
- Type：`feat`（新增）、`fix`（修复）、`refactor`（重构）、`docs`（文档）
- 示例：`feat: add Q01.10 normal matrix inverse transpose`

---

## 八、新增内容检查清单

- [ ] YAML frontmatter 格式正确且字段完整
- [ ] difficulty 为四个枚举值之一
- [ ] knowledge_points 的 id 和 name 正确
- [ ] 五步结构完整，每步内容充实
- [ ] 数学公式使用 `$` 或 `$$` 包裹，无不兼容环境
- [ ] 单字母变量使用 Markdown 斜体而非 `$...$`
- [ ] 向量统一使用 `\mathbf{}`
- [ ] 行内公式前后有空格，块级公式前后有空行
- [ ] 文件命名使用英文连字符
- [ ] 对应章节的 index.md 已更新
- [ ] 无截断行（`\` 结尾）和 Unicode 转义残留
