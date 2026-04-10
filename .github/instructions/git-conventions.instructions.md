---
applyTo: "**"
---

# Git 提交规范 + 分支/PR 工作流

## Commit 规范

- 按功能分批提交，便于审查和回溯
- Commit message 使用英文，格式：`{type}: {简短描述}`
- Type：`feat`（新增）、`fix`（修复）、`refactor`（重构）、`docs`（文档）
- 示例：`feat: add Q01.10 normal matrix inverse transpose`

---

## 分支/PR 工作流

> **核心原则**：所有变更通过分支隔离，经 PR 合并到 main，禁止直接 push main。

### 分支命名规范

```
{type}/{short-description}

示例：
feat/Q06.10-transparent-rendering
fix/single-letter-variable-rendering
docs/claude-md-split
refactor/sync-rules
```

Type 取值：`feat`、`fix`、`docs`、`refactor`

### PR 创建流程

1. 从 main 创建分支：`git checkout -b {type}/{description}`
2. 完成修改后提交：`git add -A && git commit -m "{type}: {description}"`
3. 推送分支：`git push origin {type}/{description}`
4. 通过 GitHub API 创建 PR
5. 自动合并（Squash and merge）

### PR 规范

- **标题格式**：`{type}: {简短描述}`，与 commit message 一致
- **描述模板**：自动生成，包含变更摘要
- **合并策略**：Squash and merge（保持 main 历史整洁）
- **自动合并**：创建 PR 后立即通过 API 合并（单人项目无需人工 Review）

### PR 的价值（即使单人项目）

1. **保护 main 分支**：所有变更都经过分支隔离
2. **变更记录**：PR 历史比 commit 历史更有语义
3. **可回滚**：出问题时直接 revert PR
4. **未来协作**：如果项目开放给其他人，已有 PR 流程

### 面试题生成场景的完整 Git 操作

```bash
# ① 创建分支
git checkout -b feat/Q{chapter}.{序号}-{filename}

# ② 完成所有文件修改后提交
git add -A
git commit -m "feat: add Q{chapter}.{序号} {filename}"

# ③ 推送分支
git push origin feat/Q{chapter}.{序号}-{filename}

# ④ 通过 GitHub API 创建 PR 并自动合并
# 使用 curl 调用 GitHub REST API：
# POST /repos/huziyang-henry/graphics-rendering-interview/pulls
# 然后 PUT /repos/.../pulls/{number}/merge (merge_method=squash)
```
