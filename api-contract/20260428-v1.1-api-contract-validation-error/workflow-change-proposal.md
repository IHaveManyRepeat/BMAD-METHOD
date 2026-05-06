# 工作流变更提案

## 当前问题

**问题描述：** 前后端对 `userSettings` 数据格式约定不一致，导致用户发送消息时返回 400 错误。

**具体表现：**
1. 前端发送 `platformSource: "mes"`（小写），后端期望 `"Mes"`（PascalCase）
2. 前端可能发送缺少 `tenantId` 的请求，但后端要求该字段必需
3. 前端 `UserCredentialsPayload` 类型定义与后端 Schema 不同步

**根本原因：**
- 缺乏前后端契约的单一真实来源（Single Source of Truth）
- Schema 定义分散在前后端，依赖人工同步
- 缺少数据转换层，前端直接使用存储数据发送请求

---

## 提议的变更

### 变更 1：引入数据规范化层（立即实施）

**位置：** `packages/frontend/src/chat/lib/normalize-user-settings.ts`

**目的：** 在发送 API 请求前，将前端存储的凭证格式转换为后端 API 契约要求的格式。

**实现：**
```typescript
export function normalizeUserSettingsForApi(
  credentials: UserCredentials | null
): UserCredentials | undefined {
  if (!credentials) return undefined;
  if (!credentials.tenantId?.trim()) return undefined;

  return {
    cellphone: credentials.cellphone,
    accessToken: credentials.accessToken,
    platformSource: capitalizeFirstLetter(credentials.platformSource?.toLowerCase()),
    tenantId: credentials.tenantId,
  };
}
```

**影响：**
- 修改文件：`chat-store.ts`（调用新函数）
- 新增文件：`normalize-user-settings.ts` + 测试文件
- 向后兼容：不破坏现有功能

---

### 变更 2：强化前端类型定义（立即实施）

**位置：** `packages/frontend/src/shared/lib/api-client.ts`

**目的：** 使用字面量类型约束 `platformSource`，在编译时捕获类型错误。

**变更前：**
```typescript
export interface UserCredentialsPayload {
  readonly platformSource?: string;  // ❌ 任意字符串
  // ...
}
```

**变更后：**
```typescript
export type PlatformSource = "Aigis" | "Mes" | "General";

export interface UserCredentialsPayload {
  readonly platformSource?: PlatformSource;  // ✅ 枚举约束
  // ...
}
```

**影响：**
- 修改文件：`api-client.ts`
- 可能的破坏：如果有地方使用非枚举值会编译失败（这是好事）

---

### 变更 3：改进后端错误消息（可选，建议实施）

**位置：** `packages/backend/src/routes/chat.ts`

**目的：** 提供更清晰的错误信息，帮助前端和用户快速定位问题。

**变更前：**
```typescript
message: parsed.error.issues.map((i) => i.message).join("; ")
// 输出：Invalid option: expected one of "Aigis"|"Mes"|"General"; Invalid input: expected string, received undefined
```

**变更后：**
```typescript
message: formatValidationErrors(parsed.error.issues)
// 输出：参数验证失败: platformSource: Invalid option; tenantId: Invalid input
```

**影响：**
- 修改文件：`chat.ts`
- 向后兼容：不破坏现有功能

---

### 变更 4：建立 OpenAPI 规范（长期）

**位置：** `docs/openapi/chat-api.yaml`（新建）

**目的：** 定义前后端共同遵循的 API 契约，作为单一真实来源。

**内容示例：**
```yaml
components:
  schemas:
    UserSettingsPayload:
      type: object
      properties:
        cellphone:
          type: string
          description: 用户手机号（可选）
        accessToken:
          type: string
          description: 访问令牌（可选）
        platformSource:
          type: string
          enum: [Aigis, Mes, General]
          description: 平台来源
        tenantId:
          type: string
          description: 租户 ID（必需）
      required:
        - tenantId
```

**影响：**
- 新增文件：OpenAPI 规范
- 后续步骤：使用工具自动生成前端类型
- 优先级：Medium（可在 v1.2 实现）

---

## 影响

### 代码影响

| 文件 | 变更类型 | 影响范围 |
|------|----------|----------|
| `packages/frontend/src/chat/stores/chat-store.ts` | 修改 | 低 |
| `packages/frontend/src/chat/lib/normalize-user-settings.ts` | 新增 | 中 |
| `packages/frontend/src/shared/lib/api-client.ts` | 修改 | 低 |
| `packages/backend/src/routes/chat.ts` | 修改（可选） | 低 |

### 测试影响

| 测试类型 | 新增 | 修改 |
|----------|------|------|
| 单元测试 | 2 个新测试套件 | 1 个现有测试 |
| 集成测试 | 1 个新场景 | 0 |
| E2E 测试 | 2 个新场景 | 0 |

### 风险评估

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 破坏现有功能 | 低 | 低 | 充分的单元测试 + E2E 测试 |
| 性能影响 | 极低 | 极低 | 仅增加一个简单函数调用 |
| 兼容性问题 | 低 | 中 | 向后兼容设计 |

---

## 迁移路径

### 阶段 1：立即修复（1-2 小时）

1. **创建数据规范化层**
   - 新建 `normalize-user-settings.ts`
   - 实现规范化函数
   - 编写单元测试（覆盖率 100%）

2. **集成到现有代码**
   - 修改 `chat-store.ts` 使用新函数
   - 验证现有 E2E 测试通过

3. **强化类型定义**
   - 修改 `api-client.ts` 中的 `UserCredentialsPayload`
   - 运行 `tsc --noEmit` 确保无类型错误

4. **验证修复**
   - 运行单元测试：`pnpm test`
   - 运行 E2E 测试：`pnpm test:e2e`
   - 手动测试：使用小写 "mes" 发送消息

### 阶段 2：可选改进（30 分钟）

1. **改进后端错误消息**
   - 修改 `chat.ts` 中的错误格式化逻辑
   - 添加集成测试验证错误格式

### 阶段 3：长期改进（1-2 天，v1.2 规划）

1. **定义 OpenAPI 规范**
   - 创建 `docs/openapi/chat-api.yaml`
   - 使用 Swagger UI 生成文档

2. **自动生成类型**
   - 配置 `openapi-typescript-codegen`
   - 从 OpenAPI 规范生成前端类型

3. **添加契约测试**
   - 使用 Pact 或类似工具
   - 集成到 CI/CD 流程

---

## 验收标准

### 功能验收

- [ ] 用户输入小写 "mes" 时，消息成功发送
- [ ] 用户缺失 tenantId 时，前端不发送 userSettings
- [ ] 现有 E2E 测试全部通过
- [ ] 手动测试无 400 错误

### 质量验收

- [ ] 新增代码单元测试覆盖率 ≥ 100%
- [ ] TypeScript 类型检查无错误
- [ ] Lint 检查无警告
- [ ] CI/CD 流程通过

### 文档验收

- [ ] 代码注释清晰
- [ ] 验证计划文档完整
- [ ] Bug 分析报告归档

---

*由 `bmad-analyze-bug` 工作流生成*
