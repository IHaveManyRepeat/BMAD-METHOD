# 自动化验证计划

## Bug 引用

- **Bug ID**: 20260428-v1.1-api-contract-validation-error
- **Bug 类型**: `api-contract`
- **Bug 标题**: userSettings 发送时后端验证失败导致 400 错误
- **触发日期**: 2026-04-28

---

## 1. 单元测试

### 1.1 测试用例

| # | 测试文件 | 测试名称 | 验证内容 | 优先级 |
|---|----------|----------|----------|--------|
| 1 | `packages/frontend/src/chat/lib/normalize-user-settings.test.ts` | `should capitalize platformSource` | 验证 platformSource 从小写转换为 PascalCase | P0 |
| 2 | `packages/frontend/src/chat/lib/normalize-user-settings.test.ts` | `should validate tenantId presence` | 验证 tenantId 存在时才发送 userSettings | P0 |
| 3 | `packages/frontend/src/chat/lib/normalize-user-settings.test.ts` | `should return undefined when credentials missing` | 验证凭证缺失时返回 undefined | P1 |
| 4 | `packages/frontend/src/chat/lib/normalize-user-settings.test.ts` | `should handle all platformSource values` | 验证所有平台来源的正确转换 | P1 |
| 5 | `packages/frontend/src/shared/lib/api-client.test.ts` | `sendChatSSE should use normalized userSettings` | 验证 sendChatSSE 使用规范化后的数据 | P0 |
| 6 | `packages/backend/src/routes/chat.test.ts` | `should reject lowercase platformSource` | 验证后端拒绝小写 platformSource | P1 |
| 7 | `packages/backend/src/routes/chat.test.ts` | `should reject missing tenantId` | 验证后端拒绝缺失的 tenantId | P1 |

### 1.2 测试代码示例

```typescript
// packages/frontend/src/chat/lib/normalize-user-settings.test.ts
import { describe, expect, it } from 'vitest';
import { normalizeUserSettingsForApi, type UserCredentials } from './normalize-user-settings';

describe('normalizeUserSettingsForApi', () => {
  const validCredentials: UserCredentials = {
    cellphone: '15247335755',
    accessToken: '15247335755',
    platformSource: 'mes',
    tenantId: '16'
  };

  it('should capitalize platformSource to PascalCase', () => {
    // Arrange
    const credentials = { ...validCredentials, platformSource: 'mes' as any };

    // Act
    const result = normalizeUserSettingsForApi(credentials);

    // Assert
    expect(result?.platformSource).toBe('Mes');
  });

  it('should capitalize all platformSource variants', () => {
    expect(normalizeUserSettingsForApi({ ...validCredentials, platformSource: 'aigis' as any })?.platformSource).toBe('Aigis');
    expect(normalizeUserSettingsForApi({ ...validCredentials, platformSource: 'general' as any })?.platformSource).toBe('General');
  });

  it('should validate tenantId presence', () => {
    // Arrange
    const credentialsWithoutTenantId = { ...validCredentials, tenantId: '' };

    // Act
    const result = normalizeUserSettingsForApi(credentialsWithoutTenantId);

    // Assert
    expect(result).toBeUndefined();
  });

  it('should return undefined when credentials is null', () => {
    // Act
    const result = normalizeUserSettingsForApi(null);

    // Assert
    expect(result).toBeUndefined();
  });

  it('should preserve optional fields when present', () => {
    // Act
    const result = normalizeUserSettingsForApi(validCredentials);

    // Assert
    expect(result).toEqual({
      cellphone: '15247335755',
      accessToken: '15247335755',
      platformSource: 'Mes',
      tenantId: '16'
    });
  });
});
```

---

## 2. 集成测试

| # | 场景 | 文件 | 验证内容 |
|---|------|------|----------|
| 1 | 完整聊天流程（带凭证） | `e2e/chat-with-credentials.spec.ts` | 验证使用有效凭证发送消息成功 |
| 2 | 聊天流程（凭证大小写不正确） | `e2e/chat-invalid-platformsource.spec.ts` | 验证前端自动修正大小写 |
| 3 | 聊天流程（缺失 tenantId） | `e2e/chat-missing-tenantid.spec.ts` | 验证前端不发送 userSettings |
| 4 | 后端验证测试 | `packages/backend/src/integration/chat-validation.spec.ts` | 验证后端 Schema 验证逻辑 |

---

## 3. E2E 测试

| # | 流程 | 步骤 | 验证内容 |
|---|------|------|----------|
| 1 | 用户设置凭证并发送消息 | 8 步 | 验证完整流程无 400 错误 |
| 2 | 使用历史凭证自动发送 | 5 步 | 验证 localStorage 中的凭证正确加载和使用 |

**E2E 测试流程示例：**
```typescript
// e2e/chat-with-credentials.spec.ts
import { test, expect } from '@playwright/test';

test('user can send message with valid credentials', async ({ page }) => {
  // 1. 打开应用
  await page.goto('/');

  // 2. 打开设置面板
  await page.click('[data-testid="settings-button"]');

  // 3. 填写凭证（使用小写 mes 测试自动转换）
  await page.fill('[data-testid="input-cellphone"]', '15247335755');
  await page.fill('[data-testid="input-access-token"]', '15247335755');
  await page.selectOption('[data-testid="select-platform-source"]', 'mes');
  await page.fill('[data-testid="input-tenant-id"]', '16');

  // 4. 保存凭证
  await page.click('[data-testid="save-credentials-button"]');
  await expect(page.locator('[data-testid="toast-success"]')).toBeVisible();

  // 5. 关闭设置面板
  await page.click('[data-testid="close-settings-button"]');

  // 6. 输入消息
  await page.fill('[data-testid="chat-input"]', '我有几块地？');

  // 7. 发送消息
  await page.click('[data-testid="send-button"]');

  // 8. 验证消息成功发送（无错误提示）
  await expect(page.locator('[data-testid="message-bubble-user"]')).toContainText('我有几块地？');
  await expect(page.locator('[data-testid="error-banner"]')).not.toBeVisible();
});
```

---

## 4. 静态分析

### 4.1 Lint 规则

添加 TypeScript 规则，确保 `UserCredentialsPayload` 的 `platformSource` 使用字面量类型而非 `string`：

```typescript
// packages/frontend/src/shared/lib/api-client.ts
export type PlatformSource = "Aigis" | "Mes" | "General";

export interface UserCredentialsPayload {
  readonly cellphone?: string;
  readonly accessToken?: string;
  readonly platformSource?: PlatformSource;  // 使用字面量类型而非 string
  readonly tenantId: string;
}
```

### 4.2 类型检查

- [ ] 确保前端 `UserCredentialsPayload.platformSource` 为枚举类型
- [ ] 确保后端 `UserSettingsPayloadSchema` 与前端类型一致
- [ ] 运行 `tsc --noEmit` 确保无类型错误

---

## 5. 运行时检查

### 5.1 输入验证（前端）

```typescript
// packages/frontend/src/chat/lib/normalize-user-settings.ts
import type { UserCredentials, PlatformSource } from '../../shared/lib/api-client';

/** 平台来源映射表：小写/混合 → PascalCase */
const PLATFORM_SOURCE_MAP: Record<string, PlatformSource> = {
  'aigis': 'Aigis',
  'mes': 'Mes',
  'general': 'General',
};

/**
 * 规范化用户凭证以符合 API 契约
 * - 转换 platformSource 为 PascalCase
 * - 验证 tenantId 存在
 * - 返回 undefined 如果凭证无效
 */
export function normalizeUserSettingsForApi(
  credentials: UserCredentials | null
): UserCredentials | undefined {
  // 凭证不存在
  if (!credentials) {
    return undefined;
  }

  // tenantId 是必需的
  if (!credentials.tenantId || credentials.tenantId.trim() === '') {
    console.warn('[normalizeUserSettings] tenantId is missing or empty');
    return undefined;
  }

  // 转换 platformSource 为 PascalCase
  const normalizedPlatformSource = credentials.platformSource
    ? PLATFORM_SOURCE_MAP[credentials.platformSource.toLowerCase()] ?? credentials.platformSource as PlatformSource
    : undefined;

  return {
    cellphone: credentials.cellphone,
    accessToken: credentials.accessToken,
    platformSource: normalizedPlatformSource,
    tenantId: credentials.tenantId,
  };
}
```

### 5.2 输出验证（后端）

后端已有 Zod Schema 验证，无需额外修改。建议改进错误消息：

```typescript
// packages/backend/src/routes/chat.ts
// 改进错误消息，分别显示每个字段的问题
const formatValidationErrors = (issues: z.ZodIssue[]): string => {
  const fieldErrors = issues
    .map(issue => {
      const field = issue.path.join('.');
      return `${field}: ${issue.message}`;
    })
    .join('; ');
  return `参数验证失败: ${fieldErrors}`;
};

// 使用方式
if (!parsed.success) {
  return c.json(
    {
      ok: false,
      error: {
        code: "VALIDATION_ERROR",
        message: formatValidationErrors(parsed.error.issues),
        details: parsed.error.issues,  // 额外提供详细错误
      },
    },
    400,
  );
}
```

---

## 6. CI 集成

### 6.1 Pipeline Gate

```yaml
# .github/workflows/test.yml (或对应 CI 配置)
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: pnpm install

      - name: Run type check
        run: pnpm tsc --noEmit

      - name: Run linter
        run: pnpm lint

      - name: Run unit tests
        run: pnpm test

      - name: Run integration tests
        run: pnpm test:integration

      - name: Run E2E tests
        run: pnpm test:e2e
```

### 6.2 验证脚本

```bash
#!/bin/bash
# scripts/verify-api-contract.sh

echo "🔍 验证前后端 API 契约一致性..."

# 检查 UserCredentialsPayload 类型定义
if ! grep -q "platformSource.*Aigis.*Mes.*General" packages/frontend/src/shared/lib/api-client.ts; then
  echo "❌ 前端 platformSource 类型定义不正确"
  exit 1
fi

# 检查后端 Schema 定义
if ! grep -q 'enum.*\["Aigis".*"Mes".*"General"\]' packages/backend/src/routes/chat.ts; then
  echo "❌ 后端 platformSource 枚举定义不正确"
  exit 1
fi

echo "✅ API 契约验证通过"
```

---

## 7. 测试覆盖率要求

| 模块 | 最低覆盖率 | 目标覆盖率 |
|------|------------|------------|
| `normalize-user-settings.ts` | 100% | 100% |
| `api-client.ts` (sendChatSSE) | 90% | 95% |
| `chat.ts` (后端验证逻辑) | 85% | 90% |

---

*由 `bmad-analyze-bug` 工作流生成*
