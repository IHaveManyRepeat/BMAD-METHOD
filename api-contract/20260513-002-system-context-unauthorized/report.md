# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260513-002 |
| **Title** | 用户端偏好设置无法读取管理端系统上下文字段定义 |
| **Type** | api-contract |
| **Severity** | HIGH |
| **Status** | open |
| **Analyzed At** | 2026-05-13T18:00:00.000Z |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 001 |

---

## 1. Bug Description

### 1.1 Summary

在管理端创建/配置系统上下文字段后，用户端设置组件 `PreferencesForm` 无法加载这些字段定义进行偏好设置。原因是用户端 `preferences.ts` 通过 HTTP 调用管理端 API 获取 schema 时，**管理端 `system_context_router` 要求 JWT 认证**，但用户端调用时没有携带任何认证凭证。

### 1.2 Reproduction Steps

1. 在管理端（admin-frontend）访问「系统上下文」管理页面
2. 创建一个字段，例如：`user_region`（string 类型）
3. 打开用户端（packages/frontend）设置面板 → 系统偏好区域
4. 观察偏好设置区域显示"暂无可配置的偏好字段"或警告"无法同步字段定义"
5. 检查浏览器 Network 面板：`GET /api/preferences/schema` 返回 **401 Unauthorized**

### 1.3 Expected Behavior

用户端调用 `GET /api/preferences/schema` 时能够获取到管理端配置的字段定义列表，以便用户进行偏好设置。

### 1.4 Actual Behavior

用户端从 `preferences.ts` 的 `fetchSchemaFromAdmin()` 调用 `http://localhost:3001/api/admin/system-context-fields?per_page=100`，响应 **401 Unauthorized**。`cachedSchema` 为 `null`，前端显示"暂无可配置的偏好字段"。

---

## 2. Root Cause Analysis

### 2.1 Call Chain

```
PreferencesForm (mounted)
  └─> useEffect → load()
       ├─> getPreferenceSchema()
       |    └─> apiClient("/preferences/schema")
       |         └─> fetch("http://localhost:3001/api/preferences/schema")
       |              └─> preferences.ts GET handler
       |                   └─> fetchSchemaFromAdmin()
       |                        └─> fetch("http://localhost:3001/api/admin/system-context-fields?per_page=100")
       |                             ⚠️ 401 Unauthorized — no JWT token
       |
       └─> getPreferences()
            └─> OK — this queries user_preferences table directly (no auth required)
```

### 2.2 Authentication Gap

| Component | File | Auth Required? | Has Token? |
|-----------|------|---------------|------------|
| admin-backend `system_context_router` | `admin-backend/src/features/system_context/routes.rs:17-33` | JWT required via `Extension<Claims>` | NO |
| user-backend `preferences.ts` | `packages/backend/src/routes/preferences.ts:51-71` | NO — direct DB query | N/A |
| user-backend `fetchSchemaFromAdmin()` | `packages/backend/src/routes/preferences.ts:51` | YES — calls admin API | **NO** |

### 2.3 Code Evidence

**admin-backend/routes.rs:17-22** — JWT 保护生效：
```rust
pub fn system_context_router() -> Router<SystemContextState> {
    Router::new().route(
        "/api/admin/system-context-fields",
        get(list_fields_handler).post(create_field_handler),  // _claims: Extension<Claims> in every handler
    )
```

**preferences.ts:51-60** — 无认证调用：
```typescript
async function fetchSchemaFromAdmin(adminApiUrl: string): Promise<ContextField[]> {
    const url = `${adminApiUrl}/api/admin/system-context-fields?per_page=100`;
    const response = await fetch(url, {
        method: "GET",
        headers: { "Content-Type": "application/json" },
        signal: AbortSignal.timeout(5000),
        // ❌ No Authorization header
    });
```

---

## 3. Affected Code Files

| File | Role |
|------|------|
| `packages/backend/src/routes/preferences.ts` | BUG ORIGIN — `fetchSchemaFromAdmin()` 缺少 JWT token |
| `packages/frontend/src/chat/components/PreferencesForm.tsx` | 显示层 — 接收到空 schema 时显示"暂无可配置" |
| `packages/frontend/src/chat/api/preferencesApi.ts` | API 接口定义 |
| `packages/backend/src/bootstrap.ts:70` | `adminApiUrl` 配置（默认 `http://localhost:3001`）|
| `admin-backend/src/features/system_context/routes.rs` | 管理端路由定义（正确要求 JWT，非 bug） |

---

## 4. Proposed Fix

### 4.1 Root Fix: Add Service-to-Service JWT

在用户端后端（user-backend）调用管理端 API 时，需要携带有效的 JWT token。

**问题**：目前 `preferences.ts` 的 `createPreferencesRoutes` 只接收 `db` 和 `adminApiUrl`，没有 JWT token。

**解决**：通过 `ADMIN_JWT_SERVICE_TOKEN` 环境变量配置服务间通信 token，在 `fetchSchemaFromAdmin()` 中携带 `Authorization: Bearer <token>` header。

### 4.2 Implementation Plan

1. **admin-backend**: 新增 service token 生成/验证接口，或允许配置静态 service token
2. **packages/backend/src/routes/preferences.ts**: 
   - 新增 `adminServiceToken` 配置项
   - `fetchSchemaFromAdmin()` 添加 `Authorization: Bearer <token>` header
3. **packages/backend/src/bootstrap.ts**: 传递 `ADMIN_SERVICE_TOKEN` 到 preferences routes
4. **packages/frontend/src/chat/components/PreferencesForm.tsx**: 保持不变（已正确调用 `/api/preferences/schema`）

### 4.3 Alternative: 绕过认证读取（不推荐）

admin-backend 可以新增一个无需认证的公开 endpoint 读取字段列表，用于服务间同步。但这会降低安全性，且与管理端的安全设计冲突。

---

## 5. Workflow Recommendation

**Bug Type**: `api-contract`  
**Fix Path**: `story` — 需要跨两个服务的修改，且涉及认证机制变更

建议创建 Story：`用户端偏好设置同步管理端系统上下文字段（服务间认证）`

---

*Report created via bmad-analyze-bug workflow.*