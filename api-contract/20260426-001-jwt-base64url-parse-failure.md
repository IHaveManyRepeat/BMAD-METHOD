# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | `20260426-001` |
| **Title** | JWT Base64URL 解析失败 — 登录成功但不跳转 Dashboard |
| **Type** | `api-contract` — 前后端数据编码格式不一致 |
| **Severity** | CRITICAL |
| **Status** | fixed |
| **Analyzed At** | 2026-04-26 |
| **Fixed At** | 2026-04-26 |
| **Project** | diy-a2ui |
| **Version** | v1.2 |

---

## 1. Bug Description

### 1.1 Summary

前端 `parseJwt` 使用浏览器原生 `atob()` 解码 JWT payload，但 JWT 规范（RFC 7519）使用 Base64URL 编码。`atob()` 遇到 Base64URL 的 `-` 或 `_` 字符直接抛 DOMException，被 `catch` 静默吞掉，导致 `isTokenValid()` 永远返回 `false`，`ProtectedRoute` 永远渲染登录页。

### 1.2 Reproduction Steps

1. 启动 admin-backend (Rust/Axum) 和 admin-frontend (Vite/React)
2. 访问 `http://localhost:5174/login`
3. 输入账号密码，点击登录
4. 浏览器 Network 面板显示 `POST /api/admin/auth/login` 返回 200，body 包含 `access_token`
5. **页面不跳转**，停留在登录页（`ProtectedRoute` 判定未认证）

### 1.3 Expected Behavior

登录成功后，`useLogin.onSuccess` 将 token 写入 localStorage，`navigate('/')` 跳转 Dashboard。`ProtectedRoute` 通过 `useAuthCheck()` → `isTokenValid()` 验证 token 有效后渲染 Dashboard 布局。

### 1.4 Actual Behavior

`isTokenValid()` 内部 `parseJwt()` 对 Base64URL 编码的 JWT 调用 `atob()`，抛异常后返回 `null`，`isTokenValid()` 返回 `false`。`ProtectedRoute` 判定未认证，在 `/` 路径渲染 `LoginPage` 而非 Dashboard。

---

## 2. Code Graph Analysis

### 2.1 Affected Code Paths

```
admin-frontend/src/features/auth/hooks/useAuth.ts           (BUG 所在 — parseJwt/atob)
admin-frontend/src/features/auth/hooks/useAuth.ts           (isTokenValid — 依赖 parseJwt)
admin-frontend/src/features/auth/hooks/useAuth.ts           (useAuthCheck — 调用 isTokenValid)
admin-frontend/src/App.tsx                                  (ProtectedRoute — 调用 useAuthCheck)
admin-frontend/src/features/auth/hooks/useAuth.ts           (useLogin.onSuccess — 写 localStorage)
admin-frontend/src/features/auth/__tests__/useAuth.test.tsx  (测试盲区 — 用 btoa 构造 token)
admin-frontend/e2e/login.spec.ts                            (E2E 盲区 — 仅检查 URL)
admin-frontend/e2e/helpers/database.ts                      (loginAs — 绕过 parseJwt)
```

### 2.2 Call Graph Hotspots

| Node | Connections | Complexity | Role |
|------|-------------|------------|------|
| `useAuth.ts:16` parseJwt | 3 | HIGH | 核心缺陷点 — Base64URL 与 Base64 不兼容 |
| `useAuth.ts:32` isTokenValid | 2 | LOW | 依赖 parseJwt 返回值 |
| `useAuth.ts:40` useAuthCheck | 1 | LOW | 认证状态入口 |
| `App.tsx` ProtectedRoute | 1 | LOW | 路由守卫 |

### 2.3 Dependency Chain

```
useLogin.onSuccess → localStorage.setItem('access_token', token) → navigate('/')
  → ProtectedRoute renders → useAuthCheck() → isTokenValid(localStorage.getItem('access_token'))
    → parseJwt(token) → atob(base64url_string) → DOMException!
      → catch: return null → isTokenValid: return false → ProtectedRoute: show LoginPage
```

### 2.4 Visualization

**Code Graph HTML**: `_bmad-output/implementation-artifacts/bug-analysis/code-graph.html`（未生成，逻辑链路已在上文清晰描述）

---

## 3. Root Cause Analysis

### 3.1 Immediate Cause

`parseJwt()` 使用 `atob()` 解码 JWT payload 的第二段（Base64URL 编码），但 `atob()` 仅支持标准 Base64（字符集 `A-Za-z0-9+/=`），不支持 Base64URL 的 `-` 和 `_` 字符。

**失败概率计算**：对于 130 字节的典型 JWT payload（~176 个 Base64URL 字符），出现 `-` 或 `_` 的概率 ≈ 99.6%。

### 3.2 Root Cause

前端对 JWT 编码格式做了**错误假设**，假定 JWT 使用标准 Base64 编码。实际上 JWT 规范（RFC 7519 §2）明确使用 Base64URL 编码（RFC 4648 §5），两者字符集差异：

| 标准 Base64 | Base64URL |
|-------------|-----------|
| `+` | `-` |
| `/` | `_` |
| 有 `=` padding | 无 padding |

### 3.3 Workflow-Level Cause

**为什么工作流允许这个 bug 发生？**

1. **单元测试使用与生产不一致的编码**：测试用 `btoa()`（标准 Base64）构造 fake token，生产环境后端用 Base64URL 编码。测试全绿但生产全红。

2. **E2E 测试只检查 URL 不检查内容**：`login.spec.ts` 仅断言 `page.url().toContain('/')`，而 `navigate('/')` 执行成功但 `ProtectedRoute` 在 `/` 重新渲染了登录页，测试误判为通过。

3. **`loginAs` 辅助函数绕过了认证链路**：E2E helper 直接往 localStorage 注入 token，不经过 `parseJwt`，导致问题被测试辅助工具掩盖。

4. **catch 块静默吞错误**：`parseJwt` 的 `catch` 仅 `return null`，无日志输出，增加了排查难度。

### 3.4 Similar Patterns

- 前后端数据格式不一致是常见 bug 源头（snake_case vs camelCase、日期格式、编码格式等）
- "测试全绿但生产全红"的典型模式：mock/测试数据与真实数据编码不一致
- 任何涉及编解码的地方（JWT、URL encoding、HTML entities）都容易出现类似问题

---

## 4. Prevention Strategy

### 4.1 Workflow Change

| Current | Proposed |
|---------|----------|
| 单元测试用 `btoa()` 构造 fake token | 用与生产一致的 Base64URL 编码构造 fake token |
| E2E 只检查 URL 跳转 | E2E 检查页面关键内容（登录表单消失 + Dashboard 可见） |
| `loginAs` 绕过认证链路 | `loginAs` 走完整登录流程（或增加完整流程 E2E） |
| `catch` 块静默返回 `null` | `catch` 块记录 `console.warn`，提供可追踪信息 |
| 无完整用户流程 E2E | 新增 `full-user-journey.spec.ts` 覆盖登录→导航→CRUD→退出 |

### 4.2 Automated Validation

1. **前后端编码一致性检查**：单元测试中 fake JWT 必须使用 Base64URL 编码，与后端 jsonwebtoken crate 输出一致
2. **E2E 内容断言**：登录成功后必须验证 Dashboard 关键元素可见，而非仅检查 URL
3. **完整用户流程 E2E**：覆盖 登录 → 侧边栏导航 → 创建环境 → 录入接口 → 退出，确保端到端链路畅通
4. **禁止静默 catch**：所有 catch 块必须有日志输出或显式错误处理

---

## 5. Fix Proposal

### 5.1 Fix Path

**Decision**: `hotfix` — 最小化修改，仅修复 `parseJwt` 函数和测试盲区

**修改内容：**

1. **`useAuth.ts`** — 新增 `decodeBase64Url()` 辅助函数，在调用 `atob()` 前将 Base64URL 转换为标准 Base64
2. **`useAuth.test.tsx`** — 新增 `createFakeJwt()` 辅助函数，使用 Base64URL 编码构造 fake token
3. **`login.spec.ts`** — 增加登录后页面内容断言
4. **`full-user-journey.spec.ts`** — 新增完整用户流程 E2E 测试

### 5.2 Code Changes

**核心修复** (`useAuth.ts`):

```typescript
// Base64URL → UTF-8 字符串解码
function decodeBase64Url(str: string): string {
  const base64 = str.replace(/-/g, '+').replace(/_/g, '/')
  const padded = base64 + '='.repeat((4 - (base64.length % 4)) % 4)
  return atob(padded)
}

function parseJwt(token: string): TokenPayload | null {
  try {
    const parts = token.split('.')
    if (parts.length !== 3) {
      console.warn('[parseJwt] Invalid JWT structure: expected 3 parts, got', parts.length)
      return null
    }
    const payload = JSON.parse(decodeBase64Url(parts[1]))
    return payload as TokenPayload
  } catch (err) {
    console.warn('[parseJwt] Failed to decode JWT payload:', err)
    return null
  }
}
```

---

## 6. Automated Verification Mechanism

### 6.1 Unit Tests

- [x] `useAuth.test.tsx` — 使用 Base64URL 编码的 fake token 测试 `useAuthCheck`
- [x] `useAuth.test.tsx` — 过期 Base64URL token 返回 `isAuthenticated: false`
- [x] `useAuth.test.tsx` — 无效格式 token 返回 `isAuthenticated: false`

### 6.2 Integration Tests

- [x] `e2e/login.spec.ts` — 登录成功后验证 Dashboard 侧边栏可见
- [x] `e2e/login.spec.ts` — 错误密码显示错误提示

### 6.3 Static Analysis

- [ ] ESLint rule: 禁止 `atob()` 直接用于 JWT 解码（建议规则）

### 6.4 Runtime Checks

- [x] `parseJwt` catch 块输出 `console.warn` 日志
- [x] `full-user-journey.spec.ts` 覆盖完整登录→导航→退出流程

---

## 7. Impact Assessment

| Area | Impact |
|------|--------|
| Workflows affected | 登录认证、路由守卫 |
| Files modified | `useAuth.ts`, `useAuth.test.tsx`, `login.spec.ts` |
| Files created | `full-user-journey.spec.ts` |
| Tests required | 3 单元测试修复 + 1 E2E 增强 + 1 新 E2E 文件 |
| Migration needed | 否 |

---

## 8. Lessons Learned

1. **测试数据必须与生产环境编码一致**：用 `btoa()` 构造标准 Base64 fake token，而生产用 Base64URL，是"测试全绿但生产全红"的典型案例。**禁止 mock 数据与生产数据格式不一致**。

2. **E2E 测试必须验证页面内容而非仅 URL**：`navigate('/')` 成功不代表页面正确渲染。`ProtectedRoute` 在未认证时会在 `/` 渲染登录页，URL 检查无法发现此问题。

3. **测试辅助工具不能绕过被测代码**：`loginAs` 直接注入 localStorage 绕过了 `parseJwt`，使得认证链路中最关键的一环从未被 E2E 测试覆盖。

4. **catch 块禁止静默吞错误**：`return null` 不带任何日志，使得排查时完全无迹可循。至少需要 `console.warn` 级别的日志。

5. **编解码是高频 bug 源头**：JWT Base64URL vs Base64、URL encoding、HTML entities 等编解码差异是最容易被忽略的跨层问题。前后端必须明确约定编码格式。

---

*Generated by `bmad-analyze-bug` workflow*
