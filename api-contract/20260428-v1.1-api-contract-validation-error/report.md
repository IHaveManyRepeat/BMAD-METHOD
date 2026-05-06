# Bug Analysis Report

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260428-v1.1-api-contract-validation-error |
| **Title** | userSettings 发送时后端验证失败导致 400 错误 |
| **Type** | `api-contract` — 接口契约问题 |
| **Severity** | High（阻止用户正常发送消息） |
| **Status** | open |
| **Analyzed At** | 2026-04-28 |
| **Project** | diy-a2ui-master |
| **Version** | v1.1 |

---

## 1. Bug Description

### 1.1 Summary

用户发送聊天消息时，前端发送的 `userSettings` 数据格式不符合后端 Schema 验证要求，导致返回 400 Bad Request 错误。

### 1.2 Reproduction Steps

1. 用户在设置页面填写凭证，`platformSource` 保存为小写 `"mes"`（UI 可能显示 "MES 系统"但保存值为小写）
2. 用户在聊天界面输入消息并发送
3. 前端 `chat-store.ts` 直接使用 `credentials` 数据构建请求体
4. 请求发送到 `POST /api/chat`
5. 后端 `SendChatSchema` 验证失败，返回 400 错误

### 1.3 Expected Behavior

- 消息应成功发送
- 前端应自动将 `platformSource` 转换为后端期望的 PascalCase 格式
- 如果 `tenantId` 缺失，前端应不发送 `userSettings` 字段

### 1.4 Actual Behavior

- 返回 400 Bad Request 错误
- 错误消息：`"Invalid option: expected one of \"Aigis\"|\"Mes\"|\"General\"; Invalid input: expected string, received undefined"`
- 消息未发送成功

### 1.5 错误请求示例

```json
// 请求体
{
  "conversationId": "8cb77e96-64bd-464d-b819-a906213a2264",
  "content": "我有几块地？",
  "userSettings": {
    "cellphone": "15247335755",
    "accessToken": "15247335755",
    "platformSource": "mes",    // ❌ 小写，应为 "Mes"
    // ❌ tenantId 字段完全缺失
  }
}

// 响应
{
  "ok": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid option: expected one of \"Aigis\"|\"Mes\"|\"General\"; Invalid input: expected string, received undefined"
  }
}
```

---

## 2. Code Graph Analysis

### 2.1 Affected Code Paths

**前端：**
```
InputArea.tsx (用户输入)
  ↓ sendMessage()
chat-store.ts:sendMessage()
  ↓ 获取 credentials
chat-store.ts:200-209
  ↓ 构建 UserCredentialsPayload
api-client.ts:sendChatSSE()
  ↓ fetch('/api/chat')
Backend: POST /api/chat
  ↓ SendChatSchema.safeParse()
routes/chat.ts:109-121
  ↓ 验证失败
400 Bad Request
```

**后端：**
```
routes/chat.ts:88 (router.post('/chat'))
  ↓
routes/chat.ts:100-107 (解析 JSON)
  ↓
routes/chat.ts:109-121 (SendChatSchema 验证)
  ↓ UserSettingsPayloadSchema
routes/chat.ts:22-27
  - platformSource: enum["Aigis","Mes","General"]  ❌ 期望 PascalCase
  - tenantId: string (required)  ❌ 字段缺失
```

### 2.2 Call Graph Hotspots

| Node | Connections | Complexity | Role |
|------|-------------|------------|------|
| `chat-store.ts:sendMessage()` | 4 | 中 | 核心业务逻辑，直接使用 credentials |
| `api-client.ts:sendChatSSE()` | 2 | 低 | HTTP 请求发送 |
| `routes/chat.ts:POST /api/chat` | 3 | 中 | API 入口，Schema 验证 |

### 2.3 Dependency Chain

```
输入消息
  → chat-store.sendMessage()
     → credentials (from settings-store)
        → UserCredentialsPayload (api-client.ts)
           → fetch('/api/chat')
              → 后端 SendChatSchema 验证
                 → ❌ 验证失败
```

### 2.4 Visualization

**Code Graph HTML**: 未生成（手动分析）

---

## 3. Root Cause Analysis

### 3.1 Immediate Cause

| 问题 | 位置 | 直接原因 |
|------|------|----------|
| `platformSource` 大小写错误 | `chat-store.ts:200-209` | 直接使用 `credentials` 数据，未转换大小写 |
| `tenantId` 缺失 | `chat-store.ts:200-209` | 数据构建时未验证字段存在性 |
| 错误提示模糊 | `routes/chat.ts:116` | 多个错误消息合并，不易定位 |

### 3.2 Root Cause

**核心问题：前后端 Schema 缺乏单一真实来源（Single Source of Truth）**

1. **Schema 定义分散且不一致**
   - 前端 `UserCredentialSchema`（存储/显示）：所有字段必填
   - 前端 `UserCredentialsPayload`（API 传输）：所有字段可选，`platformSource` 无枚举约束
   - 后端 `UserSettingsPayloadSchema`（API 接收）：仅 `tenantId` 必填，`platformSource` 有枚举约束

2. **数据转换层缺失**
   - 前端直接使用存储的 `credentials` 数据发送 API 请求
   - 没有中间层进行格式规范化（大小写转换、字段补充）

3. **类型约束不足**
   - 前端 `UserCredentialsPayload.platformSource` 类型为 `string`，未限制为枚举值
   - 编译时无法捕获类型错误

### 3.3 Workflow-Level Cause

**为什么工作流允许这个 bug 发生？**

| 工作流环节 | 缺失的机制 |
|------------|------------|
| Design-First | 缺少 OpenAPI/TypeSpec 契约定义 |
| 开发 | 缺少契约代码自动生成工具（如 openapi-typescript） |
| 测试 | 缺少契约测试（如 Pact）验证前端请求符合后端 Schema |
| 代码审查 | 未检查 Schema 一致性和类型定义对齐 |

### 3.4 Similar Patterns

此 bug 属于常见的 API 契约不一致问题，类似模式可能在其他 API 端点也存在：
- `canvas/submit` 端点
- 任何涉及 `userSettings` 传递的端点

建议对整个项目进行契约一致性审查。

---

## 4. Prevention Strategy

### 4.1 Workflow Change

| Current | Proposed |
|---------|----------|
| Schema 定义分散在前后端 | 引入 OpenAPI 规范作为单一真实来源 |
| 手动维护类型定义 | 使用工具自动生成前端类型 |
| 直接使用存储数据发送 API | 添加数据规范化层 |
| 错误消息模糊 | 提供清晰的字段级别错误提示 |

### 4.2 Automated Validation

- **短期**：在 `chat-store.ts` 添加数据规范化函数，确保发送前格式正确
- **中期**：使用 `openapi-typescript-codegen` 自动生成前端类型
- **长期**：引入契约测试（Pact）验证前后端一致性

---

## 5. Fix Proposal

### 5.1 Fix Path

**Decision**: `story` — 创建 story 进行修改，包含验证测试

**原因**：
- 涉及前端多个文件修改
- 需要新增单元测试和 E2E 测试
- 虽然改动量不大，但需要完整的测试覆盖

### 5.2 Validation Plan

See: [`./validation-plan.md`](./validation-plan.md)

### 5.3 Workflow Change Proposal

See: [`./workflow-change-proposal.md`](./workflow-change-proposal.md)

---

## 6. Automated Verification Mechanism

### 6.1 Unit Tests

- [ ] `packages/frontend/src/chat/lib/normalize-user-settings.test.ts` — 验证数据规范化逻辑
- [ ] `packages/frontend/src/shared/lib/api-client.test.ts` — 验证 sendChatSSE 使用规范化数据
- [ ] `packages/backend/src/routes/chat.test.ts` — 验证后端 Schema 验证

### 6.2 Integration Tests

- [ ] `e2e/chat-with-credentials.spec.ts` — 完整聊天流程（带凭证）
- [ ] `e2e/chat-invalid-platformsource.spec.ts` — 验证前端自动修正大小写
- [ ] `e2e/chat-missing-tenantid.spec.ts` — 验证缺失 tenantId 时的行为

### 6.3 Static Analysis

- [ ] 前端 `UserCredentialsPayload.platformSource` 改为枚举类型
- [ ] 运行 `tsc --noEmit` 确保无类型错误

### 6.4 Runtime Checks

- [ ] 实现 `normalizeUserSettingsForApi()` 函数
- [ ] 在发送请求前调用规范化函数

---

## 7. Impact Assessment

| Area | Impact |
|------|--------|
| Workflows affected | 聊天消息发送 |
| Files modified | `chat-store.ts`（修改）、`normalize-user-settings.ts`（新增）、`api-client.ts`（修改）、`chat.ts`（可选修改） |
| Tests required | 7 个单元测试、3 个 E2E 测试 |
| Migration needed | 否（向后兼容） |
| Deployment risk | 低（仅前端改动，后端可选） |

---

## 8. Lessons Learned

1. **API 契约必须单一真实来源**：前后端 Schema 分散定义必然导致不一致
2. **类型约束是第一道防线**：使用枚举类型而非 `string` 可以在编译时捕获错误
3. **数据转换层必不可少**：内部存储格式与 API 传输格式应通过转换层隔离
4. **清晰的错误消息能加速调试**：合并多个错误消息会增加定位难度
5. **契约测试防止回归**：自动化测试验证前后端契约一致性

---

## 9. Quick Fix Reference

### 立即修复（1-2 小时）

**文件 1: 新增 `packages/frontend/src/chat/lib/normalize-user-settings.ts`**

```typescript
import type { UserCredentials, PlatformSource } from '../../shared/lib/api-client';

const PLATFORM_SOURCE_MAP: Record<string, PlatformSource> = {
  'aigis': 'Aigis',
  'mes': 'Mes',
  'general': 'General',
};

export function normalizeUserSettingsForApi(
  credentials: UserCredentials | null
): UserCredentials | undefined {
  if (!credentials) return undefined;
  if (!credentials.tenantId?.trim()) return undefined;

  return {
    cellphone: credentials.cellphone,
    accessToken: credentials.accessToken,
    platformSource: credentials.platformSource
      ? PLATFORM_SOURCE_MAP[credentials.platformSource.toLowerCase()] ?? credentials.platformSource as PlatformSource
      : undefined,
    tenantId: credentials.tenantId,
  };
}
```

**文件 2: 修改 `packages/frontend/src/chat/stores/chat-store.ts:200-209`**

```typescript
// 修改前
const credentials = useSettingsStore.getState().credentials;
const userSettings: UserCredentialsPayload | undefined = credentials
  ? {
      cellphone: credentials.cellphone,
      accessToken: credentials.accessToken,
      platformSource: credentials.platformSource,
      tenantId: credentials.tenantId,
    }
  : undefined;

// 修改后
import { normalizeUserSettingsForApi } from '../lib/normalize-user-settings';

const credentials = useSettingsStore.getState().credentials;
const userSettings = normalizeUserSettingsForApi(credentials);
```

**文件 3: 修改 `packages/frontend/src/shared/lib/api-client.ts`（可选但推荐）**

```typescript
export type PlatformSource = "Aigis" | "Mes" | "General";

export interface UserCredentialsPayload {
  readonly cellphone?: string;
  readonly accessToken?: string;
  readonly platformSource?: PlatformSource;  // 使用枚举类型
  readonly tenantId: string;
}
```

---

*Generated by `bmad-analyze-bug` workflow*
