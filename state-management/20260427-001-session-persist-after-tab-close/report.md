# Bug Analysis Report

**Bug ID**: 20260427-001-session-persist-after-tab-close
**日期**: 2026-04-27
**严重级别**: High
**Bug 类型**: state-management（状态管理问题）
**状态**: 已修复

---

## 1. Bug 描述

### 现象

登录管理端后，关闭浏览器标签，再次打开页面，自动进入登录后的 Dashboard 页面，但 Sidebar 底部显示「未登录」。

### 复现步骤

1. 访问 `/login`，输入账号密码登录
2. 确认进入 Dashboard，Sidebar 显示用户名
3. 关闭浏览器标签
4. 重新打开页面 URL
5. 页面自动进入 Dashboard，但 Sidebar 显示「未登录」

### 预期行为

关闭标签后登录状态自动失效，再次打开页面应显示登录页。

### 实际行为

自动进入 Dashboard 页面，但账户显示未登录。

---

## 2. 代码调用图

```
LoginPage.tsx
  └─→ useLogin() [useAuth.ts]
        ├─→ loginApi() → POST /api/admin/auth/login
        └─→ onSuccess:
              ├─→ localStorage.setItem('access_token')      ← 直接存储 ①
              ├─→ localStorage.setItem('refresh_token')     ← 直接存储 ②
              ├─→ localStorage.setItem('token_expires_at')  ← 直接存储 ③
              ├─→ useAuthStore.setAuth()                    ← Zustand persist ④
              │     └─→ localStorage['auth-storage'] = {user, accessToken, refreshToken}
              └─→ navigate('/')

App.tsx
  └─→ ProtectedRoute
        └─→ useAuthCheck()
              ├─→ localStorage.getItem('access_token')  ← 从 ① 读取
              └─→ isTokenValid() → parseJwt() → 检查 exp

AppLayout.tsx
  └─→ useAuthStore() → {user, logout}
        └─→ Sidebar(userName={user?.username})

apiClient.ts
  ├─→ 请求拦截器: localStorage.getItem('access_token')
  └─→ 响应拦截器: localStorage.getItem('refresh_token')
```

### 🔴 双重存储反模式

```
存储层 A (直接 localStorage):
  ├─ 'access_token'
  ├─ 'refresh_token'
  └─ 'token_expires_at'

存储层 B (Zustand persist):
  └─ 'auth-storage' = {user, accessToken, refreshToken}
```

---

## 3. 根因分析

### 3.1 直接原因

`localStorage` 持久化存储在标签关闭后**不会清除**。浏览器重新打开时：
1. `useAuthCheck()` 从 `localStorage['access_token']` 读到有效 JWT → `isAuthenticated = true`
2. 路由放行进入 Dashboard
3. `AppLayout` 从 `useAuthStore` 读取 `user` → Zustand persist 的水合是**异步**的，首次渲染时 `user = null`
4. Sidebar 显示 `userName ?? '未登录'` → **显示「未登录」**

### 3.2 根本原因（3 层）

| 层级 | 原因 | 位置 |
|------|------|------|
| **L1: 存储类型错误** | 使用 `localStorage` 而非 `sessionStorage`，跨会话持久化 | `useAuthStore.ts`, `useAuth.ts`, `apiClient.ts` |
| **L2: 双重存储反模式** | Token 同时存在直接 localStorage 和 Zustand persist 两处，无单一数据源 | `useAuth.ts:61-64` vs `useAuthStore.ts:13-28` |
| **L3: 水合竞态** | Zustand persist 的水合是异步的，首次渲染拿不到 user | `App.tsx:15` vs `AppLayout.tsx:30` |

### 3.3 工作流层面根因

开发时没有建立「认证状态单一数据源 + 会话级存储」的架构约束，导致各组件自行从 `localStorage` 直接读写 token。

---

## 4. 修复方案

### 策略：统一到 Zustand + sessionStorage

| 文件 | 变更 | 原则 |
|------|------|------|
| `useAuthStore.ts` | persist 改用 `sessionStorage`；增加 `getAccessToken`/`getRefreshToken` 外部 getter；增加 `setTokens` 方法 | SRP: store 是唯一数据源 |
| `useAuth.ts` | 移除所有直接 `localStorage.setItem/getItem`；`useAuthCheck` 等待水合完成 | DRY: 去除重复存储 |
| `apiClient.ts` | 拦截器通过 getter 读取 token，refresh 后通过 `store.setTokens()` 更新 | DIP: 依赖抽象 getter |

### 修复效果

- ✅ 关闭标签 → `sessionStorage` 自动清除 → 重新打开进入登录页
- ✅ Zustand store 是唯一数据源 → user 和 token 始终同步
- ✅ 外部 getter 解决 apiClient 非 React 上下文读取问题
- ✅ 水合回调同步外部 getter → 无竞态窗口

---

## 5. 影响范围

### 修改文件

| 文件 | 变更行数 |
|------|----------|
| `admin-frontend/src/features/auth/hooks/useAuthStore.ts` | +48 / -7 |
| `admin-frontend/src/features/auth/hooks/useAuth.ts` | +30 / -15 |
| `admin-frontend/src/shared/lib/apiClient.ts` | +12 / -10 |
| `admin-frontend/src/features/auth/__tests__/useAuth.test.tsx` | 重写 |
| `admin-frontend/src/features/auth/__tests__/LoginPage.test.tsx` | 重写 |

### 未修改文件

- `LoginPage.tsx` — 无需修改（通过 hook 间接操作 store）
- `App.tsx` — 无需修改（`ProtectedRoute` 使用 `useAuthCheck`）
- `AppLayout.tsx` — 无需修改（通过 `useAuthStore` 读取）
- `Sidebar.tsx` — 无需修改（通过 props 接收 userName）
- 后端 — 无需修改

---

## 6. 验证结果

- ✅ 23 个单元测试全部通过
- ✅ TypeScript 编译零错误（认证相关文件）
- ✅ sessionStorage persist 验证通过
- ✅ localStorage 不被写入验证通过
- ✅ 外部 getter 同步验证通过
