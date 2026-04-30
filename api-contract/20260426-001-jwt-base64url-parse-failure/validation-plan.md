# Validation Plan — JWT Base64URL 编码一致性

## 目标

确保前端 JWT 解析逻辑与后端 JWT 编码格式（Base64URL）永远一致，从测试链路上消除"测试全绿但生产全红"的风险。

## 验证方案

### 方案：Base64URL 一致性测试矩阵

#### 1. 单元测试层：编码格式对齐

**问题**：原测试用 `btoa()`（标准 Base64）构造 fake token，生产后端用 Base64URL 编码。

**修复**：新增 `createFakeJwt()` 辅助函数，输出 Base64URL 编码的 fake token。

```typescript
// useAuth.test.tsx
function createFakeJwt(payload: Record<string, unknown>): string {
  const jsonStr = JSON.stringify(payload)
  const base64 = btoa(jsonStr)
  // 标准 Base64 → Base64URL：替换字符，去掉 padding
  const base64Url = base64.replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
  const header = btoa(JSON.stringify({ alg: 'HS256', typ: 'JWT' }))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
  return `${header}.${base64Url}.fake-signature`
}
```

**覆盖场景**：

| 测试用例 | Token 编码 | 期望结果 |
|----------|-----------|----------|
| 有效 Base64URL token（未过期） | Base64URL | `isAuthenticated: true` |
| 过期 Base64URL token | Base64URL | `isAuthenticated: false` |
| 无效格式 token | 纯文本 | `isAuthenticated: false` |
| 无 token | null | `isAuthenticated: false` |

#### 2. E2E 测试层：内容断言替代 URL 断言

**问题**：原 E2E 只检查 `page.url().toContain('/')`，`ProtectedRoute` 在 `/` 渲染登录页也能通过。

**修复**：登录后断言 Dashboard 关键元素可见。

```typescript
// login.spec.ts
// 验证登录表单消失
await expect(page.getByText('管理端登录')).not.toBeVisible({ timeout: 5_000 })
// 验证 Dashboard 布局渲染成功（侧边栏可见）
await expect(page.locator('nav, [data-component="sidebar"], aside').first()).toBeVisible({ timeout: 5_000 })
```

#### 3. 完整流程 E2E：端到端覆盖

新增 `full-user-journey.spec.ts` 覆盖：

```
登录 → Dashboard 渲染 → 侧边栏导航（6个页面）→ 创建环境 → 录入接口 → 退出
```

**核心断言**：

| Flow | 关键断言 |
|------|---------|
| Flow 1: 登录 | URL 变为 `/` + 登录表单消失 + aside 可见 |
| Flow 2: 导航 | 点击每个菜单项 → URL 变化 + 页面标题可见 |
| Flow 3: 创建环境 | 填表提交 → 新环境名出现在列表 |
| Flow 4: 录入接口 | 填表提交 → 新接口别名出现在列表 |
| Flow 5: 退出 | 跳转到 /login + 直接访问 / 被拦截 |

#### 4. 编解码工具函数单元测试

新增独立测试文件 `__tests__/base64url.test.ts`：

```typescript
describe('decodeBase64Url', () => {
  it('正确解码 Base64URL 字符串（含 - 和 _）', ...)
  it('正确解码标准 Base64 字符串', ...)
  it('正确处理无 padding 的输入', ...)
  it('对非法字符抛出异常', ...)
})
```

## 验证清单

- [x] 单元测试使用 Base64URL 编码构造 fake JWT token
- [x] E2E 登录测试验证页面内容而非仅 URL
- [x] 完整用户流程 E2E 覆盖登录→导航→CRUD→退出
- [x] `parseJwt` catch 块输出 `console.warn` 日志
- [ ] 独立 `decodeBase64Url` 函数单元测试
- [ ] ESLint 规则禁止 `atob()` 直接用于 JWT 解码
