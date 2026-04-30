# Workflow Change Proposal — 测试数据生产一致性 & E2E 内容断言

## 变更概述

从工作流层面杜绝"测试全绿但生产全红"的问题，建立**测试数据生产一致性**和**E2E 内容断言**两条硬性规则。

## 变更内容

### 1. 测试数据与生产环境编码一致性

**问题**：单元测试用 `btoa()`（标准 Base64）构造 fake JWT，生产后端用 Base64URL 编码。编码格式不同导致测试永远通过，但生产永远失败。

**规则**：

> **禁止 mock/测试数据与生产数据格式不一致。** 凡涉及编解码（JWT、URL encoding、HTML entities）、序列化格式（JSON、XML）、命名风格（snake_case vs camelCase）、日期格式等，测试中使用的 fake 数据必须与生产环境输出完全一致。

**检查机制**：

| 检查点 | 执行时机 | 工具 |
|--------|---------|------|
| Code Review 时审查测试数据构造方式 | PR 合并前 | 人工审查 |
| 测试辅助函数集中管理，禁止内联构造 | CI / Lint | ESLint / 自定义规则 |
| 后端接口契约中明确编码格式 | 设计阶段 | OpenAPI spec |

**落地措施**：

1. 创建项目级测试工具库 `test-utils/jwt-factory.ts`，统一 fake JWT 构造逻辑，使用 Base64URL 编码
2. `createFakeJwt()` 成为唯一合法的 token 构造入口，禁止测试中直接使用 `btoa()` 构造 JWT
3. 后端 OpenAPI spec 中明确标注 JWT 使用 Base64URL 编码

### 2. E2E 测试内容断言标准

**问题**：E2E 登录测试仅检查 URL，不检查页面内容。`navigate('/')` 成功但 `ProtectedRoute` 在 `/` 渲染登录页，测试误判为通过。

**规则**：

> **E2E 测试必须验证页面关键内容，而非仅检查 URL。** 页面跳转断言 = URL 变化 + 关键元素可见性 + 前页面元素消失。测试辅助函数（如 `loginAs`）不能绕过被测系统的核心逻辑。

**断言模式**：

```typescript
// ✅ 正确：内容断言
await expect(page.getByText('管理端登录')).not.toBeVisible()
await expect(page.locator('aside')).toBeVisible()

// ❌ 错误：仅 URL 断言
expect(page.url()).toContain('/')
```

**落地措施**：

1. 创建 E2E 断言工具函数 `assertPageRendered(page, expectedElements)`
2. 现有 `loginAs` helper 保留用于测试前置条件，但必须补充完整登录流程 E2E（不使用 `loginAs`）
3. `full-user-journey.spec.ts` 成为 CI 必跑测试

### 3. catch 块错误处理标准

**问题**：`parseJwt` 的 `catch` 块静默返回 `null`，无任何日志，排查时无迹可循。

**规则**：

> **catch 块禁止静默吞错误。** 所有 catch 块必须至少包含 `console.warn` 或 `console.error` 级别的日志输出，附带足够的上下文信息用于排查。

**落地措施**：

1. Code Review 检查清单新增：catch 块是否有日志
2. ESLint 规则建议：检测空 catch 块或仅 `return null` 的 catch 块

### 4. 测试辅助函数规范

**问题**：`loginAs` 直接往 localStorage 注入 token，绕过了 `parseJwt` 验证链路。

**规则**：

> **测试辅助函数不能绕过被测系统的核心验证逻辑。** 如果辅助函数为了性能绕过某些步骤，必须另外有独立的 E2E 测试覆盖被绕过的完整链路。

**落地措施**：

1. `loginAs` 用于需要已认证状态的测试前置条件（合理）
2. `full-user-journey.spec.ts` 覆盖完整登录→认证→导航流程（不使用 `loginAs`）
3. 两者互补，不替代

## 实施优先级

1. **P0**（已完成）：`parseJwt` Base64URL 修复 + 单元测试编码对齐
2. **P0**（已完成）：E2E 登录测试内容断言增强
3. **P0**（已完成）：`full-user-journey.spec.ts` 完整用户流程覆盖
4. **P1**：创建 `test-utils/jwt-factory.ts` 统一 token 构造
5. **P1**：创建 `assertPageRendered` E2E 断言工具
6. **P2**：ESLint 规则禁止空 catch 块
7. **P2**：ESLint 规则禁止 `atob()` 直接用于 JWT 解码
8. **P3**：后端 OpenAPI spec 中明确 JWT Base64URL 编码格式
