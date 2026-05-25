# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260514-004 |
| **Title** | Preferences API 500 — 跨 schema 查询架构设计缺陷 |
| **Type** | workflow-flow |
| **Severity** | HIGH |
| **Status** | open |
| **Analyzed At** | 2026-05-14T10:00:00.000Z |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 004 |

---

## 1. Bug Description

### 1.1 Summary

调用 `GET /api/preferences` 接口时返回 500 Internal Server Error，SQL 错误显示 `relation "user_test"."user_preferences" does not exist`。问题的根本原因是**架构设计缺陷**：`user_preferences` 表位于 user schema，但管理端凭证（`ADMIN_SERVICE_TOKEN`）和 schema 配置未正确设置，导致用户端后端无法正确访问数据库。

用户提出的正确架构方向：**用户端调用管理端接口**查询系统上下文列表，凭证写入用户端后端配置，这样兼顾安全（不暴露管理端凭证给前端）、不移动表、不考虑跨域。

### 1.2 Reproduction Steps

1. 启动用户端后端服务（packages/backend）
2. 前端调用 `GET http://localhost:5173/api/preferences`
3. 接口返回 500 错误，耗时 4132ms
4. 服务端日志：`Failed query: select "field_name", "field_value" from "user_test"."user_preferences"`

### 1.3 Expected Behavior

- `GET /api/preferences` 应正常返回用户的偏好设置
- 系统上下文字段定义应从管理端获取
- 凭证应安全存储在后端配置中

### 1.4 Actual Behavior

- 接口返回 HTTP 500 错误
- 错误响应：`{"ok":false,"error":{"code":"INTERNAL_ERROR","message":"Failed query: select \"field_name\", \"field_value\" from \"user_test\".\"user_preferences\"\nparams: ","traceId":"..."}}`

---

## 2. Affected Areas

### 2.1 Affected Workflows

- `bmad-pipeline` — 部署/启动流程

### 2.2 Affected Files

| 文件 | 问题描述 |
|------|----------|
| `packages/backend/src/routes/preferences.ts` | 直接查询 user schema 的 `user_preferences` 表 |
| `packages/backend/src/shared/config.ts` | `adminServiceToken` 和 `adminApiUrl` 未正确配置 |
| `admin-backend/src/features/system_context/routes.rs` | 已有 `/api/admin/system-context-fields/public` 接口 |
| `.env` (用户端) | `ADMIN_SERVICE_TOKEN` 可能未配置或为空 |

### 2.3 当前架构问题

```
前端 → GET /api/preferences → 用户端后端 → 查询 user_test.user_preferences
                                   ↑
                            表不存在或配置错误

正确架构应该是：
前端 → GET /api/preferences → 用户端后端 → 调用管理端 API (/api/admin/system-context-fields/public)
                                   ↓
                            X-Service-Token 认证
                                   ↓
                            返回字段定义 + 查询用户偏好表
```

---

## 3. Root Cause Analysis

### 3.1 代码流程分析

```
GET /api/preferences
  → createPreferencesRoutes() [packages/backend/src/routes/preferences.ts:186]
  → deps.db.select({ fieldName, fieldValue }).from(userPreferences)
  → userPreferences 来自 getSchemaInstance().user
  → getSchemaInstance() 返回 pgSchema("user_test").table("user_preferences")
  → 生成的 SQL: SELECT "field_name", "field_value" FROM "user_test"."user_preferences"
  → 错误：relation does not exist
```

### 3.2 问题分类

根据 `config.yaml` 的 bug type 定义，此问题属于 **workflow-flow**（工作流程设计缺陷）：

- 当前流程：用户端直接查询本地 `user_preferences` 表
- 问题：表可能不存在于配置的 schema 中
- 正确流程：用户端通过 service-to-service 调用管理端 API 获取字段定义和偏好数据

### 3.3 架构缺陷分析

根据用户描述，正确架构应该是：

1. **凭证存储在用户端后端**：不暴露给前端
2. **通过管理端 API 查询系统上下文字段列表**：使用 `/api/admin/system-context-fields/public`
3. **不移动表**：`user_preferences` 表保留在 user schema
4. **不跨域**：同服务内部调用

### 3.4 代码图 (Code Graph)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      packages/frontend                               │
│                 GET /api/preferences (SettingsPanel)                │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼ HTTP GET
┌─────────────────────────────────────────────────────────────────────┐
│                    packages/backend (用户端)                          │
│              GET /api/preferences [preferences.ts:186]               │
│                                                                  │
│  问题: 直接查询 user_test.user_preferences 表                      │
│  表可能不存在或 schema 配置错误                                     │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼ deps.db.select()
┌─────────────────────────────────────────────────────────────────────┐
│                    PostgreSQL (user_test schema)                    │
│              "relation \"user_preferences\" does not exist"           │
└─────────────────────────────────────────────────────────────────────┘

正确架构应该是：
┌─────────────────────────────────────────────────────────────────────┐
│                    packages/backend (用户端)                          │
│              GET /api/preferences [preferences.ts:186]               │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼ fetch /api/admin/system-context-fields/public
                             │  Header: X-Service-Token: {ADMIN_SERVICE_TOKEN}
┌─────────────────────────────────────────────────────────────────────┐
│                 admin-backend (管理端)                               │
│         GET /api/admin/system-context-fields/public [L44]            │
│         验证 X-Service-Token → 返回字段定义列表                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Fix Proposal

### 4.1 根本性解决方案

按照用户提出的架构重构：

1. **管理端**：已有 `/api/admin/system-context-fields/public` 接口（L44-71 in `routes.rs`），使用 `X-Service-Token` 认证
2. **用户端**：通过 service-to-service 调用管理端 API 获取系统上下文字段定义
3. **凭证管理**：`ADMIN_SERVICE_TOKEN` 存储在用户端后端 `.env` 配置中，不暴露给前端

### 4.2 实施步骤

#### Step 1: 确认管理端 public 接口

管理端已有 `public_list_fields_handler`：
- 路径：`GET /api/admin/system-context-fields/public`
- 认证：`X-Service-Token` header 与 `ADMIN_SERVICE_TOKEN` 环境变量比对
- 返回：`FieldListResponse`（字段列表）

#### Step 2: 确认用户端配置

`packages/backend/src/shared/config.ts` 中已有：
```typescript
adminApiUrl: env.ADMIN_API_URL?.trim() ?? "http://localhost:3001",
adminServiceToken: env.ADMIN_SERVICE_TOKEN?.trim() ?? "",
```

#### Step 3: 验证用户端调用管理端

`packages/backend/src/routes/preferences.ts` 中的 `fetchSchemaFromAdmin` 函数已经正确实现：
```typescript
async function fetchSchemaFromAdmin(adminApiUrl: string, adminServiceToken: string): Promise<ContextField[]> {
  const url = `${adminApiUrl}/api/admin/system-context-fields/public`;
  const response = await fetch(url, {
    method: "GET",
    headers: {
      "Content-Type": "application/json",
      "X-Service-Token": adminServiceToken,
    },
    signal: AbortSignal.timeout(5000),
  });
  // ...
}
```

### 4.3 验证命令

1. 确认管理端服务运行中
2. 确认 `ADMIN_SERVICE_TOKEN` 在用户端 `.env` 中配置
3. 测试管理端接口：
```bash
curl -H "X-Service-Token: YOUR_SERVICE_TOKEN" \
     http://localhost:3001/api/admin/system-context-fields/public
```
4. 测试用户端接口：
```bash
curl http://localhost:5173/api/preferences
```

---

## 5. Validation Plan

1. 确认管理端 `ADMIN_SERVICE_TOKEN` 环境变量已设置
2. 确认 `ADMIN_API_URL` 指向正确的管理端地址
3. 重启用户端后端服务
4. 调用 `GET /api/preferences/schema` 验证字段定义获取
5. 调用 `GET /api/preferences` 验证偏好查询

---

## 6. Related Bugs

- `20260514-003` — preferences API 返回 500 错误
- `20260514-001` — 用户端凭证设置未读取管理端系统上下文配置的字段
- `20260513-002` — system context unauthorized

---

*Bug analysis document created. This is a workflow-flow issue requiring architectural fix.*