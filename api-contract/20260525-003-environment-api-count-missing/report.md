# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260525-003 |
| **Title** | 环境管理页接口数量未展示数据 |
| **Type** | api-contract — API 契约问题 |
| **Severity** | Medium |
| **Status** | fixed |
| **Analyzed At** | 2026-05-25 |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 003 |

---

## 1. Bug Description

### 1.1 Summary

环境管理页面的"接口数量"列始终显示为 `—`（破折号），未展示真实数据。

### 1.2 Reproduction Steps

1. 打开管理后台 → 环境管理页面
2. 观察列表中任意环境的"接口数量"列
3. 始终显示 `—`，无论该环境下实际关联了多少个 API

### 1.3 Expected Behavior

"接口数量"列应显示该环境下已注册的 API 数量（如 0、3、12 等）。

### 1.4 Actual Behavior

"接口数量"列始终显示 `—` 占位符。

---

## 2. Code Graph Analysis

### 2.1 Affected Code Paths

```
admin-backend/src/features/environment/model.rs     — EnvironmentResponse 缺少 api_count 字段
admin-backend/src/features/environment/repo.rs       — find_all 未关联 api_registrations 表
admin-backend/src/features/environment/service.rs    — list_environments 未计算 api_count
admin-frontend/src/features/environments/types/index.ts — Environment 类型缺少 api_count
admin-frontend/src/features/environments/components/EnvironmentList.tsx — 硬编码 &mdash; 占位符
```

### 2.2 Call Graph

```
Frontend:
  EnvironmentList.tsx → useEnvironments() → environmentApi.list() → GET /api/admin/environments
    └─ <TableCell>&mdash;</TableCell>  ← 硬编码，未读取任何数据

Backend:
  GET /environments → routes.rs → service::list_environments()
    → repo::find_all() → SELECT * FROM environments (无 api_count)
    → EnvironmentResponse::from_environment() (无 api_count 参数)
    → JSON Response (无 api_count 字段)
```

### 2.3 Dependency Chain

```
前端占位符 (根因呈现层)
  ← 前端类型定义缺失 api_count 字段
    ← 后端 API 响应未包含 api_count
      ← EnvironmentResponse 结构体未定义 api_count
        ← service 层未查询 api_registrations 计数
          ← repo 层查询未 JOIN api_registrations 表
```

---

## 3. Root Cause Analysis

### 3.1 Immediate Cause

前端 `EnvironmentList.tsx:124` 硬编码 `<TableCell>&mdash;</TableCell>`，未读取任何环境字段来展示接口数量。

### 3.2 Root Cause

**前后端契约双重缺失：**

1. **后端** `EnvironmentResponse` 结构体不包含 `api_count` 字段。`list_environments` 的 SQL 查询仅 `SELECT * FROM environments`，未与 `api_registrations` 表关联。`from_environment` 转换方法也没有接收 count 参数的入口。

2. **前端** `Environment` 类型定义未声明 `api_count` 字段。即使后端添加了该字段，TypeScript 类型也不会识别。

3. 前端开发时用了 `&mdash;` 作为临时占位符，但后续从未对接真实数据源。

### 3.3 Workflow-Level Cause

**为什么工作流允许这个 bug 发生？**

- 缺乏前后端接口一致性验证：前端 UI 列定义了"接口数量"列，但没有自动化机制检查后端响应是否包含对应字段。
- 占位符模式未被追踪：`&mdash;` 这类占位符在代码审查中容易被忽略，没有 lint 规则或 grep 脚本扫描此类临时代码。
- Story 完成标准未覆盖"所有列均有真实数据"的验证要求。

### 3.4 Similar Patterns

类似模式曾在项目中出现：
- `20260515-002-unregistered-skills-type-mismatch` — 前后端类型不匹配
- `20260506-005-test-proxy-422-contract-mismatch` — 前后端契约不一致

建议关注：搜索整个前端代码库中其他 `&mdash;` 或 `—` 占位符，确认是否存在类似的未完成功能。

---

## 4. Prevention Strategy

### 4.1 Workflow Change

| Current | Proposed |
|---------|----------|
| 前端占位符 `&mdash;` 无追踪机制 | 代码审查时 grep `&mdash;` 和 `TODO`，标记为阻塞项 |
| 前后端类型独立维护 | 前端 TypeScript 类型从 OpenAPI schema 自动生成，确保一致性 |
| 无接口字段覆盖率检查 | 前端表格列定义与后端 response schema 做自动化对比 |

### 4.2 Automated Validation

- **Lint 规则**: 添加 ESLint 规则检测 JSX 中的硬编码占位符 (`&mdash;`, `TBD`, `TODO`)
- **类型生成**: 使用 `openapi-typescript` 从后端 OpenAPI spec 自动生成前端类型
- **集成测试**: 验证 `/api/admin/environments` 响应包含 `api_count` 字段且值 >= 0

---

## 5. Fix Proposal

### 5.1 Fix Path

**Decision**: `plan` — 5 个文件，约 40 行改动，直接修复无需创建 story。

### 5.2 Fix Details

| 层 | 文件 | 改动 |
|---|---|---|
| 后端 Model | `admin-backend/src/features/environment/model.rs` | `EnvironmentResponse` 增加 `api_count: i64`；`from_environment` 接受 count 参数 |
| 后端 Repo | `admin-backend/src/features/environment/repo.rs` | 新增 `count_apis_per_environment()` 批量查询 + `count_apis_for_environment()` 单条查询 |
| 后端 Service | `admin-backend/src/features/environment/service.rs` | `list_environments` 合并 api_counts HashMap；`get_environment` 查询单条 count |
| 前端 Type | `admin-frontend/src/features/environments/types/index.ts` | `Environment` 接口增加 `api_count: number` |
| 前端 UI | `admin-frontend/src/features/environments/components/EnvironmentList.tsx` | `&mdash;` → `{env.api_count}` |

**性能考量**：列表查询使用 `GROUP BY` 批量一次查出所有环境的 count（2 条 SQL，非 N+1）。单条查询复用已有的 `SELECT COUNT(*) WHERE environment_id = $1`。

---

## 6. Automated Verification Mechanism

### 6.1 Unit Tests

- [x] `model::tests::from_environment_converts_correctly` — 验证 `from_environment(&env, 3)` 返回 `api_count: 3`
- [x] `model::tests::environment_response_serializes_with_snake_case` — 验证 JSON 包含 `"api_count"` 字段

### 6.2 Integration Tests

- [ ] 验证 `GET /api/admin/environments` 响应中每个 environment 对象包含 `api_count` 字段
- [ ] 验证 `api_count` 值与 `api_registrations` 表中对应 environment_id 的 COUNT(*) 一致

### 6.3 Static Analysis

- [ ] 建议添加 ESLint 规则检测 JSX 中 `&mdash;` 硬编码占位符

### 6.4 Runtime Checks

- [x] `cargo check` 通过 — 0 错误
- [x] `cargo test` 通过 — 31 个环境相关测试全部通过

---

*Bug Analysis Report completed on 2026-05-25. Fix applied and verified.*
