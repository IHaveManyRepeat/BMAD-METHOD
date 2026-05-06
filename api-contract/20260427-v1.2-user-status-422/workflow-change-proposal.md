# Workflow Change Proposal: API 契约一致性

## 当前问题

### 问题描述

管理端用户禁用功能失败，由于前后端 API 契约不一致：

| 问题 | 描述 |
|------|------|
| 前端调用错误端点 | 使用 `PUT /api/admin/users/{id}` 禁用用户 |
| 请求体不匹配 | 发送 `{ is_active: false }`，后端期望 `{ username: "..." }` |
| 正确端点未使用 | `PATCH /api/admin/users/{id}/status` 存在但未被调用 |
| 类型定义不同步 | 前端 `UpdateUserRequest` 包含 `is_active`，后端不包含 |

### 根本原因

1. **缺乏 Design-First 流程**
   - 前后端各自独立开发
   - 没有统一的 API 规范作为"契约"
   - 没有自动化机制验证接口一致性

2. **测试覆盖不足**
   - E2E 测试未包含用户禁用场景
   - 缺少 API 契约验证测试

3. **文档不同步**
   - 前后端类型定义未集中管理
   - API 使用指南缺失

---

## 提议变更

### 方案 A：紧急修复（当前采用）

**目标**：立即修复用户禁用功能

**变更：**
1. 前端添加 `updateStatus()` API 方法
2. 前端修正 `UpdateUserRequest` 类型定义
3. 添加 E2E 测试覆盖

**优点：**
- 快速恢复功能
- 改动最小
- 不影响现有功能

**缺点：**
- 未解决根本问题（接口契约一致性）
- 未来可能再次出现类似问题

### 方案 B：Design-First 流程（推荐）

**目标**：建立长期的 API 契约一致性机制

**变更：**
1. 采用 OpenAPI 3.x 规范
2. 从 OpenAPI 生成前后端类型
3. CI 中添加契约验证

**优点：**
- 从根本上解决接口不一致问题
- 自动化类型同步
- 提升开发体验

**缺点：**
- 需要较大架构调整
- 学习曲线
- 需要额外工具

---

## 详细实施计划

### Phase 1: 紧急修复（1-2 天）

**任务 1.1：修正前端 API 调用**

- [ ] 在 `userApi.ts` 添加 `updateStatus()` 方法
- [ ] 调用 `PATCH /users/${id}/status` 端点
- [ ] 传递 `{ is_active: boolean }` 请求体

**文件：** `admin-frontend/src/features/users/api/userApi.ts`

**任务 1.2：修正前端类型定义**

- [ ] 修改 `UpdateUserRequest` 为 `{ username: string }`
- [ ] 添加 `UpdateUserStatusRequest` 接口
- [ ] 确保与后端一致

**文件：** `admin-frontend/src/features/users/types/index.ts`

**任务 1.3：补充 E2E 测试**

- [ ] 添加用户禁用测试用例
- [ ] 添加用户启用测试用例
- [ ] 验证错误处理

**文件：** `admin-frontend/e2e/users.spec.ts`

### Phase 2: 测试覆盖（2-3 天）

**任务 2.1：添加集成测试**

- [ ] 测试 `PATCH /api/admin/users/{id}/status` 成功场景
- [ ] 测试禁用超级管理员返回 403
- [ ] 测试禁用不存在的用户返回 404

**文件：** `admin-backend/tests/integration/user_tests.rs`（新增）

**任务 2.2：提升测试覆盖率**

- [ ] 确保 API 端点 100% 覆盖
- [ ] 确保 Service 层 > 80% 覆盖
- [ ] 运行 `cargo llvm-cov --fail-under-lines 80`

### Phase 3: 长期改进（可选，1-2 周）

**任务 3.1：建立 OpenAPI 规范**

```
docs/openapi/admin-api.yaml
├── paths/
│   ├── /users/
│   │   ├── get.yaml
│   │   ├── post.yaml
│   │   ├── {id}/
│   │   │   ├── get.yaml
│   │   │   ├── put.yaml
│   │   │   ├── delete.yaml
│   │   │   └── status/patch.yaml  ← 用户状态更新
│   │   └── {id}/password/patch.yaml
└── components/
    └── schemas/
        ├── User.yaml
        ├── CreateUserRequest.yaml
        ├── UpdateUserRequest.yaml
        └── UpdateUserStatusRequest.yaml
```

**任务 3.2：自动化类型生成**

```yaml
# .github/workflows/generate-types.yml
name: Generate API Types

on:
  push:
    paths:
      - 'docs/openapi/**'

jobs:
  generate-frontend:
    runs-on: ubuntu-latest
    steps:
      - name: Generate TypeScript Types
        run: |
          npx @openapitools/openapi-generator-cli generate \
            -i docs/openapi/admin-api.yaml \
            -g typescript-axios \
            -o admin-frontend/src/shared/api/generated

  generate-backend:
    runs-on: ubuntu-latest
    steps:
      - name: Generate Rust Types
        run: |
          cargo install openapi-generator
          cargo run --bin openapi-generator \
            --input docs/openapi/admin-api.yaml \
            --output admin-backend/src/shared/api \
            --language rust
```

**任务 3.3：添加契约验证**

```yaml
# .github/workflows/verify-contract.yml
name: Verify API Contract

on: [pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - name: Check Contract Consistency
        run: |
          python scripts/verify-api-contract.py \
            --openapi docs/openapi/admin-api.yaml \
            --frontend admin-frontend/src/features/**/types/index.ts \
            --backend admin-backend/src/features/**/model.rs
```

---

## 影响分析

### 正面影响

| 方面 | 改进 |
|--------|--------|
| 功能可用性 | 用户禁用/启用功能正常工作 |
| 开发体验 | 类型定义一致，减少手动同步错误 |
| 测试覆盖 | 关键用户流程有 E2E 测试保护 |
| 文档质量 | API 契约文档化，降低沟通成本 |

### 风险评估

| 风险 | 可能性 | 影响 | 缓解措施 |
|--------|----------|------|----------|
| 后端不兼容类型变更 | 低 | 中 | 严格版本控制，先改后端 |
| 测试环境不稳定 | 中 | 低 | 使用独立的测试数据库 |
| 部署延迟 | 低 | 低 | 分阶段发布 |

### 兼容性

- **向后兼容**：是，现有 API 端点不受影响
- **数据迁移**：无需，数据库结构不变
- **依赖更新**：无需，仅内部代码变更

---

## 迁移路径

### 方式 A：渐进式（推荐）

```
Week 1: 紧急修复
  ├─ 修复前端 API 调用
  ├─ 添加 E2E 测试
  └─ 发布 Hotfix

Week 2-3: 测试覆盖
  ├─ 补充集成测试
  ├─ 提升覆盖率
  └─ 建立测试基准

Week 4+: 架构改进
  ├─ 创建 OpenAPI 规范
  ├─ 配置自动生成
  └─ 持续优化
```

### 方式 B：一次性（激进）

```
一次完成所有变更：
  ├─ 修复当前 bug
  ├─ 补充所有测试
  ├─ 建立 OpenAPI 规范
  └─ 配置自动化

风险：变更量大，测试周期长
```

---

## 成功标准

### 必须达成

1. ✅ 用户禁用功能正常工作
2. ✅ 用户启用功能正常工作
3. ✅ E2E 测试全部通过
4. ✅ 测试覆盖率 > 80%

### 期望达成

1. ⚠️ OpenAPI 规范已建立
2. ⚠️ 类型自动生成已配置
3. ⚠️ CI 中有契约验证

---

## 后续行动

### 短期（1 周内）

- [ ] 完成 Phase 1 所有任务
- [ ] 修复发布到生产环境
- [ ] 监控错误日志确认问题解决

### 中期（1 个月内）

- [ ] 完成 Phase 2 所有任务
- [ ] 建立测试覆盖率基准
- [ ] 评估是否采用 Phase 3

### 长期（3 个月内）

- [ ] 完整采用 Design-First 流程
- [ ] API 契约文档化
- [ ] 自动化类型生成和验证

---

## 相关文档

- `report.md` — Bug 分析报告
- `validation-plan.md` — 验证计划
- `admin-backend/src/features/user/model.rs` — 后端类型定义
- `admin-frontend/src/features/users/api/userApi.ts` — 前端 API 调用
