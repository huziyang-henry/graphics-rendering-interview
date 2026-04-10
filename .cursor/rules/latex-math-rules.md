# LaTeX 数学公式编写规范（Cursor 专用）

本文件是 Cursor 编辑器的专用规则，与根目录 `CLAUDE.md` 配合使用。
`CLAUDE.md` 包含完整项目规范，本文件仅补充 LaTeX 公式的详细约定。

## 1. 符号速查表

### 向量（`\mathbf{}`）

| 符号 | 含义 | LaTeX |
|------|------|-------|
| n | 表面法线 | `\mathbf{n}` |
| l | 入射光方向 | `\mathbf{l}` |
| v | 观察方向 | `\mathbf{v}` |
| h | 半程向量 | `\mathbf{h}` |
| r | 反射向量 | `\mathbf{r}` |
| m | 微表面法线 | `\mathbf{m}` |
| P, V, M | 矩阵 | `\mathbf{P}`, `\mathbf{V}`, `\mathbf{M}` |

### 标量

| 符号 | 含义 | LaTeX |
|------|------|-------|
| α | 粗糙度参数 | `\alpha` |
| θ | 角度 | `\theta` |
| π | 圆周率 | `\pi` |
| σ | 标准差 | `\sigma` |
| F₀ | 基础反射率 | `F_0` |
| k_d, k_s | 漫反射/镜面反射系数 | `k_d`, `k_s` |

### 函数名（`\text{}`）

`\text{roughness}`, `\text{albedo}`, `\text{metallic}`, `\text{normalize}`, `\text{reflect}`, `\text{irradiance}`, `\text{specular}`, `\text{diffuse}`

## 2. GitHub 不兼容环境

| 禁止 | 替代方案 |
|------|---------|
| `\begin{aligned}` | `$$...$$` 内用 `\\` 换行 |
| `\begin{pmatrix}` | `\begin{bmatrix}` |
| `\begin{equation}` | `$$...$$` |
| `\tag{}` | 文字标注 |
| `\label{}` / `\ref{}` | 文字引用 |
| `\begin{matrix}` | `\begin{bmatrix}` |

## 3. 单字母变量陷阱

`$n$`、`$f$`、`$k$`、`$D$` 等单字母行内公式在 GitHub 上**不渲染**。
→ 使用 Markdown 斜体 `*n*` 或放入更完整的公式中。

## 4. 格式要求

- 行内 `$` 前后留半角空格：`能量守恒 $k_d + k_s \leq 1$，其中...`
- 块级 `$$` 前后各留一个空行
- GLSL/HLSL 代码块内容不转 LaTeX

## 5. 经验沉淀

发现新的 LaTeX 相关问题时，更新本文件。包括但不限于：
- GitHub 不支持的新 LaTeX 环境
- 渲染异常的公式写法（如单字母变量不渲染）
- 新的符号约定或格式优化
