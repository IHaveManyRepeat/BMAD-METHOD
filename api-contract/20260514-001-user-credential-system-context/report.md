# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260514-001 |
| **Title** | 用户端凭证设置未读取管理端系统上下文配置的字段 |
| **Type** | api-contract |
| **Severity** | HIGH |
| **Status** | open |
| **Analyzed At** | 2026-05-14T00:00:00.000Z |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 001 |

---

## 1. Bug Description

### 1.1 Summary

用户端（packages/frontend）的 SettingsPanel 中"用户凭证"区域的表单字段（手机号、访问令牌、平台来源、租户ID等）是在前端硬编码的，与管理端（admin-backend）`system_context_fields` 表中定义的字段没有关联。当管理端管理员配置了新的系统上下文字段时，用户端无法动态读取这些字段定义来渲染表单。

### 1.2 Reproduction Steps

1. 在管理端访问「系统上下文」页面，查看已配置的字段列表
2. 注意到字段包括：`cellphone`、`access_token`、`tenant_id`、`area_code` 等用户凭证相关字段
3. 打开用户端设置面板 → 用户凭证区域
4. 观察表单字段是否与管理端定义一致
5. **实际行为**：用户端凭证表单字段是硬编码的（SettingsPanel.tsx 手动定义 7 个字段）
6. **期望行为**：用户端应从 `/api/preferences/schema` 读取管理端配置的字段定义，动态渲染表单

### 1.3 Expected Behavior

用户端「用户凭证」表单应该：
1. 调用 `GET /api/preferences/schema` 获取管理端的系统上下文字段定义
2. 根据字段类型（string/number/boolean/select）动态渲染表单控件
3. 管理端修改字段配置后，用户端自动同步更新（无需重新部署）

### 1.4 Actual Behavior

- SettingsPanel.tsx 中的 CredentialFormState 是**硬编码**的 7 个字段
- `PreferencesForm` 组件虽然调用了 `getPreferenceSchema()` 读取系统上下文，但只用于"系统偏好"区域
- "用户凭证"区域完全独立，没有读取管理端字段定义
- 管理端在 `system_context_fields` 表中定义了字段，但用户端没有使用这个数据

---

## 2. Root Cause Analysis

### 2.1 Call Chain

```
管理端 admin-backend
  └─> system_context_fields 表 (数据库)
       └─> 字段定义: cellphone, access_token, tenant_id, area_code, ...
           └─> /api/admin/system-context-fields/public (已添加 service token 认证)

用户端 packages/frontend
  ├─> SettingsPanel.tsx (用户凭证表单)
  │    └─> CredentialFormState 硬编码 7 个字段 (cellphone, accessToken, ...)
  │         ❌ 没有读取 system_context_fields
  │
  └─> PreferencesForm.tsx (系统偏好表单)
       └─> getPreferenceSchema() → /api/preferences/schema
            └─> fetchSchemaFromAdmin(adminApiUrl, adminServiceToken)
                 └─> GET /api/admin/system-context-fields/public
                      ✅ 已正确实现 service token 认证
```

### 2.2 代码证据

**SettingsPanel.tsx:38-46** — 硬编码凭证字段：
```typescript
interface CredentialFormState {
    cellphone: string;
    accessToken: string;
    platformSource: PlatformSource;
    tenantId: string;
    areaCode: string;
    longitude: string;
    latitude: string;
}
```

**PreferencesForm.tsx:18-22** — 动态读取字段定义：
```typescript
export function PreferencesForm({ onSaved }: PreferencesFormProps) {
    const [schema, setSchema] = useState<ContextField[]>([]);
    // ...
    useEffect(() => {
        async function load() {
            const [schemaResult] = await Promise.allSettled([
                getPreferenceSchema(),  // ✅ 调用 /api/preferences/schema
            ]);
            // ...
        }
    }, []);
}
```

**preferences.ts:51-58** — 已实现 service token 认证：
```typescript
async function fetchSchemaFromAdmin(adminApiUrl: string, adminServiceToken: string): Promise<ContextField[]> {
    const url = `${adminApiUrl}/api/admin/system-context-fields/public`;
    const response = await fetch(url, {
        method: "GET",
        headers: {
            "Content-Type": "application/json",
            "X-Service-Token": adminServiceToken,  // ✅ 已添加
        },
        // ...
    });
}
```

### 2.3 问题分析

| 组件 | 文件 | 问题 |
|------|------|------|
| 用户凭证表单 | `SettingsPanel.tsx:38-46` | 硬编码字段，未读取 schema |
| 系统偏好表单 | `PreferencesForm.tsx` | ✅ 已正确调用 `getPreferenceSchema()` |
| 后端 schema 接口 | `preferences.ts:51-58` | ✅ 已实现 service token 认证 |

**根本原因**：
1. 管理端 `system_context_fields` 表定义了用户凭证字段（cellphone, access_token, 等）
2. `PreferencesForm` 已经实现了从管理端读取字段定义的功能
3. 但 `SettingsPanel` 中的"用户凭证"表单是独立实现的，没有复用 `PreferencesForm` 的逻辑
4. 两个表单使用的数据源完全不同：用户凭证用 localStorage + hardcoded fields，系统偏好用 API schema

---

## 3. Affected Code Files

| File | Role | 状态 |
|------|------|------|
| `packages/frontend/src/chat/components/SettingsPanel.tsx` | 用户凭证表单（硬编码） | ❌ 需要改造 |
| `packages/frontend/src/chat/components/PreferencesForm.tsx` | 系统偏好表单（API 驱动） | ✅ 已正确实现 |
| `packages/frontend/src/chat/api/preferencesApi.ts` | 偏好 API 定义 | ✅ 可复用 |
| `packages/backend/src/routes/preferences.ts` | 后端偏好路由 | ✅ 已正确实现 |
| `packages/backend/src/shared/config.ts` | 配置（含 adminServiceToken） | ✅ 已正确实现 |
| `admin-backend/src/features/system_context/routes.rs` | 管理端公开接口 | ✅ 已添加 public endpoint |

---

## 4. Proposed Fix

### 4.1 Root Fix: 重用 PreferencesForm 渲染用户凭证

**核心思路**：让 SettingsPanel 的"用户凭证"区域也通过 `getPreferenceSchema()` 读取管理端字段定义，复用 `PreferencesForm` 的渲染逻辑。

### 4.2 Implementation Plan

**Phase 1: 统一数据源**

1. 修改 `SettingsPanel.tsx`：
   - 删除硬编码的 `CredentialFormState` 和 `FieldError` 类型
   - 用户凭证区域改用 `PreferencesForm` 组件渲染
   - 通过 `onSaved` 回调更新 `settings-store` 中的凭证

2. 调整 `PreferencesForm.tsx`：
   - 支持不同区域的表单渲染（用户凭证 vs 系统偏好）
   - 用户凭证字段特殊处理：标记为"敏感字段"不显示在列表

3. 调整 `settings-store.ts`：
   - `UserCredentials` 类型从 Zod schema 动态推断
   - 保存时调用 `/api/preferences` 保存到 `user_preferences` 表

**Phase 2: 后端适配**

1. `preferences.ts` 的 `GET /preferences` 接口：
   - 读取 `user_preferences` 表获取用户保存的值
   - 返回字段定义 + 值 的组合

2. 确保 `system_context_fields` 表中包含所有用户凭证字段定义

### 4.3 Alternative: 保持分离但同步字段

如果用户凭证需要特殊处理（如敏感数据加密存储），可以：
1. 保持 `SettingsPanel` 的用户凭证表单独立
2. 但让管理端通过 `system_context_fields` 配置字段列表
3. 前端从 `/api/preferences/schema` 读取字段定义，动态渲染

---

## 5. Workflow Recommendation

**Bug Type**: `api-contract` — 前后端接口契约不一致
**Fix Path**: `story` — 涉及前端组件重构和后端接口调整

建议创建 Story：`用户端凭证表单重构为 API 驱动`

---

## 6. Related Bug Analyses

- **20260513-002-system-context-unauthorized**: 管理端 API 认证问题 → ✅ 已通过 `/public` endpoint + service token 解决
- **20260513-001-system-context-500**: 数据库表缺失 → ✅ 已通过迁移解决

---

*Bug Analysis completed via bmad-analyze-bug workflow.*