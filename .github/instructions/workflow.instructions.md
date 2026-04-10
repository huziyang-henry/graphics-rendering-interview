---
applyTo: "**"
---

# 面试题生成工作流 + 新增题目工作流

## 零、面试题生成工作流

当用户提出一个渲染面试问题时，按以下 7 步自动执行：

```
用户提问 → ①分类与编号 → ②创建分支 → ③生成回答MD → ④更新index.md → ⑤验证 → ⑥提交+PR → ⑦经验沉淀
```

### ① 分类与编号

1. 读取 `00-总纲.md` 获取 9 个章节列表
2. 根据问题内容匹配目标章节（关键词参考见下表）
3. 读取目标章节的 `index.md`，扫描当前最大 `Q` 编号，新编号 = 最大编号 + 1
4. 判断难度等级（默认 `intermediate`）
5. 生成英文连字符文件名（基于题目核心关键词）

**章节关键词匹配**：

| 关键词 | 章节 |
|--------|------|
| 管线、顶点、光栅化、MVP、深度、裁剪、blending | 01-渲染管线基础 |
| 光照、BRDF、PBR、菲涅尔、NDF、IBL、漫反射、高光 | 02-光照与着色模型 |
| 阴影、Shadow Map、CSM、PCF、PCSS、VSM、ESM | 03-阴影技术 |
| 纹理、Mipmap、材质、贴图、虚拟纹理、流式加载 | 04-材质与纹理系统 |
| 后处理、Tone Mapping、Bloom、TAA、SSAO、SSR、DOF | 05-后处理效果 |
| Render Graph、多线程、延迟渲染、Forward+、合批 | 06-渲染架构与管线设计 |
| Draw Call、GPU、性能、带宽、Wave、Shader变体 | 07-性能优化与GPU架构 |
| 光追、GI、Lumen、Nanite、NeRF、3DGS | 08-高级渲染技术与前沿方向 |
| 开放世界、LOD、大世界、系统设计 | 09-综合场景与系统设计 |

### ② 创建分支

```bash
git checkout -b feat/Q{chapter}.{序号}-{filename}
```

分支命名规范见 `git-conventions.instructions.md`。

### ③ 生成回答 MD

在目标章节目录下创建文件，写入 YAML frontmatter + 五步结构正文（结论→原理→工程→踩坑→延伸），遵循 LaTeX 规范。

### ④ 更新 index.md

在对应难度分组的表格末尾追加一行，如有新知识点则同步更新知识点表。**不修改** `00-总纲.md`。

### ⑤ 验证

- YAML frontmatter 格式正确
- `$` 配对完整，无不兼容 LaTeX 环境
- 无截断行和 Unicode 转义残留
- index.md 表格格式正确

### ⑥ 提交 + PR

```bash
git add -A
git commit -m "feat: add Q{chapter}.{序号} {filename}"
git push origin feat/Q{chapter}.{序号}-{filename}
```

然后通过 GitHub API 创建 PR 并自动合并（详见 `git-conventions.instructions.md`）。

### ⑦ 经验沉淀

按 `retrospective.instructions.md` 规范检查是否有通用经验需要更新到规范文件。

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
