# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260512-001 |
| **Title** | PUT /api/admin/apis/{id} 返回 500 Internal Server Error |
| **Type** | api-contract |
| **Severity** | High |
| **Status** | open |
| **Analyzed At** | 2026-05-12T03:43:00.000Z |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 001 |

---

## 1. Bug Description

### 1.1 Summary

调用 `PUT http://localhost:5174/api/admin/apis/1bedb94f-033f-403d-a3d9-22d6766423f4` 更新 API 注册信息时，后端返回 HTTP 500 Internal Server Error，没有任何有意义的错误信息。

### 1.2 Reproduction Steps

1. 前端打开 API 详情页面 `ApiDetailPage`
2. 点击编辑按钮，打开 `ApiForm` 对话框
3. 修改任意字段（如 api_alias、description 等）
4. 点击保存，触发 `updateApi.mutateAsync({ id, data: payload })`
5. 观察：请求以 PUT 发送到 `/api/admin/apis/{id}`
6. 后端返回：HTTP/1.1 500 Internal Server Error 287ms

### 1.3 Expected Behavior

PUT 请求应成功更新数据库中的 API 注册记录，返回更新后的 `ApiResponse`（包含 `needs_retest` 和 `affected_excluded_params` 字段）。

### 1.4 Actual Behavior

后端返回 500 Internal Server Error，没有任何有意义的错误信息。

---

## 2. Code Graph Analysis

### 2.1 Affected Code Paths

**前端调用链:**
- `ApiForm.tsx:125` → `updateApi.mutateAsync({ id: editItem.id, data: payload })`
- `useApiRegistrations.ts:35-36` → `apiRegistrationApi.update(id, data)`
- `apiRegistrationApi.ts:49-51` → `apiClient.put<ApiRegistration>('/apis/${id}', data)`
- `apiClient.ts:6` → `baseURL: '/api/admin'`，完整路径为 `/api/admin/apis/{id}`

**后端处理链:**
- `routes.rs:117-125` → `update_api_handler` 处理函数
- `service.rs:85-176` → `update_api` 服务函数
- `repo.rs:151-193` → `update` 仓储函数（执行 SQL UPDATE）

### 2.2 Key Data Flow

```
UpdateApiRequest (frontend)
  ↓ JSON { api_alias?, http_method?, path?, ... }
routes.rs:121 Json<UpdateApiRequest>
  ↓
service.rs:88 UpdateApiRequest
  ↓ validates + lock check + detects changes
  ↓ calls repo.update(...)
  ↓
repo.rs:164 SQL UPDATE with COALESCE
  ↓
RETURNING * → ApiRegistration
  ↓
UpdateApiResponseWithAffectedParams { api, needs_retest, affected_excluded_params }
```

### 2.3 Code Structure

| File | Role |
|------|------|
| `admin-frontend/src/features/api-registrations/api/apiRegistrationApi.ts` | API 客户端，PUT `/apis/${id}` |
| `admin-frontend/src/features/api-registrations/hooks/useApiRegistrations.ts` | React Query mutation hook |
| `admin-frontend/src/features/api-registrations/components/ApiForm.tsx` | 表单组件，调用 updateApi |
| `admin-backend/src/features/api/routes.rs` | HTTP PUT 路由处理器 |
| `admin-backend/src/features/api/service.rs` | 业务逻辑层 |
| `admin-backend/src/features/api/repo.rs` | 数据访问层，SQL UPDATE |
| `admin-backend/src/features/api/model.rs` | 数据模型定义 |
| `admin-backend/src/shared/error.rs` | 错误处理，500 对应 `AppError::Internal` 或 `AppError::Database` |

---

## 3. Root Cause Analysis

### 3.1 Immediate Cause

HTTP 500 错误由以下后端错误类型之一引发：
- `AppError::Internal(String)` — 代码逻辑错误
- `AppError::Database(String)` — 数据库操作失败

根据 `error.rs:79-102`，数据库错误会：
1. 记录原始错误到日志 (`tracing::error!`)
2. 若是 `23505`（唯一约束冲突）→ 返回 409 Conflict
3. 若是 `23503`（外键约束违反）→ 返回 400 Bad Request
4. 其他数据库错误 → 返回 500 Internal Server Error，**不暴露原始错误**

### 3.2 Root Cause

**初步判断：可能是 SQL UPDATE 语句中的 COALESCE 逻辑问题**

`repo.rs:164-178` 中的 UPDATE 语句使用 COALESCE 处理所有可选字段：

```sql
UPDATE api_registrations SET
    environment_id = COALESCE($1, environment_id),
    api_alias = COALESCE($2, api_alias),
    http_method = COALESCE($3, http_method),
    ...
WHERE id = $10
RETURNING *
```

**潜在问题分析：**

1. **`auth_config` 字段类型不匹配：**
   - `api_registrations.auth_config` 定义为 `JSONB`（PostgreSQL）
   - Rust 侧 `auth_config: Option<serde_json::Value>`
   - 若前端传递 `auth_config: null`（JSON null），sqlx 可能无法正确处理

2. **COALESCE 与 JSON null 的边界情况：**
   - PostgreSQL 中 `COALESCE(NULL, value)` 返回 `value`
   - 但若参数是 SQL 类型的 NULL（来自 serde_json::Value::Null），行为可能不符合预期

3. **environment_id 外键约束：**
   - `api_registrations.environment_id` REFERENCES `environments(id)`
   - 若传递的 `environment_id` 在数据库中不存在，会触发外键约束失败
   - 但外键违反应返回 400（23503），不是 500

**最可能的原因：**
当更新请求中包含 `environment_id` 字段时，COALESCE 逻辑可能导致数据库在执行时出现类型转换问题。特别是在 environment_id 被序列化为字符串 UUID 而不是二进制 UUID 时。

### 3.3 Workflow-Level Cause

**工作流设计缺陷：**

1. **缺少请求数据验证的详细日志：**
   - 当 500 错误发生时，后端只记录了 `tracing::error!`，但没有返回给前端具体的错误信息
   - 前端无法知道是哪个字段导致的错误

2. **错误信息双轨制未完全执行：**
   - 根据项目规范（错误信息双轨制），后端应返回 `userMessage`（用户友好）+ `debugMessage`（技术调试）
   - 当前 `AppError::Database` 的实现只返回通用的 "A database error occurred"，不包含调试信息

3. **缺少对 UPDATE 请求中可选字段的显式处理：**
   - 当前使用 COALESCE 无法区分"用户没传此字段"和"用户传了 null"
   - 应该使用更明确的字段更新策略

### 3.4 Similar Patterns

无直接类似模式，但存在潜在风险：
- 其他使用 COALESCE 进行部分更新的地方可能存在相同问题

---

## 4. Fix and Verification

### 4.1 Recommended Fix (Plan)

**修改 `repo.rs` 中的 `update` 函数，使用更明确的字段更新逻辑：**

```rust
pub async fn update(
    pool: &PgPool,
    id: Uuid,
    environment_id: Option<Uuid>,
    api_alias: Option<&str>,
    http_method: Option<&str>,
    path: Option<&str>,
    description: Option<&str>,
    params_schema: Option<&serde_json::Value>,
    response_schema: Option<&serde_json::Value>,
    auth_type: Option<&str>,
    auth_config: Option<&serde_json::Value>,
) -> Result<Option<ApiRegistration>, sqlx::Error> {
    // 动态构建 UPDATE 语句，只更新非 None 的字段
    let mut updates = vec![];
    let mut params: Vec<Box<dyn sqlx::Encode<'_, sqlx::Postgres> + Send + Sync>> = vec![];
    let mut param_idx = 1u32;

    if environment_id.is_some() {
        updates.push(format!("environment_id = ${}", param_idx));
        // params.push(Box::new(environment_id) as Box<dyn ...>);
        param_idx += 1;
    }
    // ... 其他字段类似处理

    if updates.is_empty() {
        // 没有字段需要更新，直接返回原记录
    }

    let sql = format!(
        "UPDATE api_registrations SET {} WHERE id = ${} RETURNING *",
        updates.join(", "),
        param_idx
    );
    // ... 执行查询
}
```

**或者使用 sqlx 的 named 参数方式，为每个字段使用 `None` 表示不更新：**

更好的方案是使用 `sqlx::query_builder::QueryBuilder` 动态构建查询。

### 4.2 Verification Plan

1. **单元测试：**
   - 测试 `repo::update` 对各种字段组合的处理
   - 测试 COALESCE 边界情况

2. **集成测试：**
   - 使用实际数据库测试 PUT 请求的各种场景
   - 验证返回的错误信息是否有用

3. **日志增强：**
   - 在 500 错误发生时，记录更详细的调试信息（尤其是开发环境）

---

## 4. Prevention Strategy

### 4.1 Workflow Change

| Current | Proposed |
|---------|----------|
| 使用 COALESCE 处理所有 UPDATE 字段，无法区分"未传"和"传了 null" | 动态构建 UPDATE 语句，只更新实际传入的字段 |

### 4.2 Automated Validation

- **单元测试**: 为 `repo::update` 添加边界情况测试，覆盖只更新一个字段、多个字段、所有字段不更新等场景
- **集成测试**: 验证 PUT 请求的各种字段组合都能正确处理

---

## 5. Fix Proposal

### 5.1 Bug Type

**`api-contract`** — API 契约问题

### 5.2 Fix Path

**Decision**: `plan` — 直接使用 /plan 修改 repo.rs 中的 update 函数

### 5.3 Validation Plan

See: [`./validation-plan.md`](./validation-plan.md)

---

## 6. Automated Verification Mechanism

### 6.1 Unit Tests

- [ ] `repo.rs` 中的 `update` 函数对 `environment_id = None` 的处理
- [ ] `repo.rs` 中的 `update` 函数对 `auth_config = serde_json::Value::Null` 的处理
- [ ] `repo.rs` 中的 `update` 函数对所有字段都不更新时的处理

### 6.2 Integration Tests

- [ ] PUT `/api/admin/apis/{id}` 只更新 api_alias 的场景
- [ ] PUT `/api/admin/apis/{id}` 更新所有字段的场景
- [ ] PUT `/api/admin/apis/{id}` 只更新 description（文本字段）的场景

### 6.3 Static Analysis

- [ ] 使用 clippy 检查可能的类型问题
- [ ] 检查 COALESCE 使用是否合理

### 6.4 Runtime Checks

- [ ] 在开发环境记录详细数据库错误日志
- [ ] 验证 500 错误返回时有足够的调试信息

---

## 5. Change Log

| Date | Version | Description |
|------|---------|-------------|
| 2026-05-12 | 001 | 初始 bug 分析报告 |

---

*Bug analysis completed.*
