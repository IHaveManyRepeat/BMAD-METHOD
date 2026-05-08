# Validation Plan — 管理端代码位置验证

## Bug ID: 20260508-002

## 1. Pre-Migration Validation（迁移前验证）

### 1.1 确认 admin-frontend 功能覆盖

- [ ] 对比 `packages/frontend/src/admin/pages/` 和 `admin-frontend/src/features/dashboard/` 的功能差异
- [ ] 确认 OverviewPage 与 Dashboard 是否功能重叠
- [ ] 列出需要新建的 feature 目录

### 1.2 依赖分析

- [ ] 分析 27 个文件的 import 依赖链
- [ ] 确认哪些组件依赖用户端特有库（如 Zustand），需替换为管理端方案（Axios + TanStack Query）
- [ ] 确认 `admin-client.ts` 的 API base URL 配置

## 2. Migration Validation（迁移中验证）

### 2.1 文件迁移完整性

- [ ] 27 个文件全部迁移到 `admin-frontend/src/features/` 对应目录
- [ ] `packages/frontend/src/admin/` 目录完全删除
- [ ] `packages/frontend/src/App.tsx` 中 admin 相关 import 和路由全部移除

### 2.2 Import 路径修正

- [ ] 所有迁移文件的 import 路径更新为管理端约定
- [ ] API 客户端从自定义 `admin-client.ts` 切换为管理端标准 Axios 方案
- [ ] UI 组件路径从 `@/shared/components/ui/` 改为 `@/components/ui/`

### 2.3 路由注册

- [ ] `admin-frontend/src/App.tsx` 中注册所有新迁移的路由
- [ ] 侧边栏导航链接指向正确的路由路径

## 3. Post-Migration Validation（迁移后验证）

### 3.1 功能验证

| 验证项 | URL | 预期 |
|-------|-----|------|
| 管理端概览 | `http://localhost:5174/` | 显示概览页面 |
| LLM 详情 | `http://localhost:5174/llm-details` | 显示 LLM 详情 |
| Skill 详情 | `http://localhost:5174/skill-details` | 显示 Skill 详情 |
| 会话质量 | `http://localhost:5174/session-quality` | 显示会话质量 |
| 链路追踪 | `http://localhost:5174/traces` | 显示追踪列表 |
| 报错分析 | `http://localhost:5174/errors` | 显示报错页面 |

### 3.2 负向验证

| 验证项 | URL | 预期 |
|-------|-----|------|
| 用户端无 admin 路由 | `http://localhost:5173/admin` | 404 或空白 |
| 用户端无 admin 路由 | `http://localhost:5173/admin/llm-details` | 404 或空白 |

### 3.3 构建验证

- [ ] `pnpm build` (monorepo 根目录) 构建成功
- [ ] `cd admin-frontend && pnpm build` 构建成功
- [ ] 无 TypeScript 编译错误
- [ ] 无 ESLint 错误

### 3.4 回归验证

- [ ] 用户端聊天功能正常
- [ ] 用户端 Canvas 渲染正常
- [ ] 管理端已有功能（API 注册、环境管理等）不受影响
