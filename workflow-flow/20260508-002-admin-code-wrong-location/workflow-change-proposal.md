# Workflow Change Proposal — 防止代码放错位置

## Bug ID: 20260508-002

## 1. Current Issue

AI Agent 在实现 story 时，可以自由创建文件到任意目录，没有自动化的路径归属验证机制。`project-boundaries.md` 作为被动文档存在，依赖 Agent 主动读取和遵守，但实际执行中经常被忽略。

## 2. Proposed Changes

### Change 1: PreToolUse Hook — 文件路径守卫

**目标：** 在文件创建时自动阻止违规路径。

**实现方案：** 在 `.claude/settings.json` 或 hooks 配置中添加 PreToolUse hook：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "command": "node scripts/check-path-boundary.js",
        "description": "验证文件创建路径是否符合项目目录所有权规则"
      }
    ]
  }
}
```

**`scripts/check-path-boundary.js` 逻辑：**

```javascript
// 核心规则
const RULES = [
  {
    pattern: /packages\/frontend\/src\/admin\//,
    message: "BLOCKED: 管理端代码禁止放在 packages/frontend/src/admin/ 下，应放在 admin-frontend/src/"
  },
  {
    pattern: /admin-frontend\/src\/.*from\s+['"]\.\.\/\.\.\/packages\//,
    message: "BLOCKED: admin-frontend 禁止引用 packages/ 的代码"
  }
];
```

### Change 2: Story 模板增加"目标目录"字段

**目标：** 在分配 story 时明确指定代码放置位置。

**修改：** Story 模板增加：

```markdown
## Implementation Constraints
- **Target Directory:** `admin-frontend/src/features/{feature-name}/`
- **Forbidden Paths:** `packages/frontend/src/admin/`
- **UI Library:** `@/components/ui/` (NOT `@/shared/components/ui/`)
- **State Management:** Axios + TanStack Query (NOT Zustand + fetch)
```

### Change 3: CI 构建时边界检查

**目标：** 在构建流水线中自动检测违规。

**实现方案：** 添加 npm script：

```json
{
  "check-boundaries": "node scripts/check-project-boundaries.js"
}
```

**检查内容：**
1. `packages/frontend/src/admin/` 目录不应存在
2. `admin-frontend/` 不应引用 `packages/` 的代码
3. 两个前端的 `package.json` 不应有交叉依赖

## 3. Impact

| Change | 影响范围 | 破坏性 |
|--------|---------|--------|
| PreToolUse Hook | AI Agent 文件创建操作 | 无破坏性，仅阻止违规操作 |
| Story 模板修改 | Story 创建流程 | 无破坏性，增加约束 |
| CI 边界检查 | 构建流水线 | 无破坏性，新增检查步骤 |

## 4. Migration Path

1. 先创建 `scripts/check-path-boundary.js` 脚本
2. 在 hooks 配置中注册 PreToolUse hook
3. 更新 Story 模板
4. 添加 `check-boundaries` npm script
5. 将边界检查加入 CI pipeline

## 5. Priority

**High** — 此问题已导致 27 个文件放错位置，且在没有防护措施的情况下会再次发生。
