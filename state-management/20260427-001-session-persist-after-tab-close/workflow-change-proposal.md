# Workflow Change Proposal — Session Persist After Tab Close

**Bug ID**: 20260427-001-session-persist-after-tab-close
**日期**: 2026-04-27

---

## 当前问题

### 问题描述

认证状态使用 `localStorage` 持久化存储，导致关闭浏览器标签后登录状态不会失效。同时存在双重存储反模式（直接 `localStorage` + Zustand `persist`），导致状态不同步。

### 具体表现

- `useAuthCheck` 从 `localStorage['access_token']` 读取（同步）
- `AppLayout` 从 `useAuthStore` 读取 `user`（异步水合）
- 两者数据源不同，导致认证判断和用户显示不一致

---

## 提议变更

### 变更 1：统一存储为 sessionStorage

**Before:**
```typescript
// useAuthStore.ts — 使用默认 localStorage
persist(storeConfig, { name: 'auth-storage' })

// useAuth.ts — 直接写入 localStorage
localStorage.setItem('access_token', response.access_token)
localStorage.setItem('refresh_token', response.refresh_token)
localStorage.setItem('token_expires_at', String(Date.now() + response.expires_in * 1000))
```

**After:**
```typescript
// useAuthStore.ts — 使用 sessionStorage
persist(storeConfig, {
  name: 'auth-storage',
  storage: createJSONStorage(() => sessionStorageAdapter),
  onRehydrateStorage: () => (state) => {
    if (state?.accessToken) _externalAccessToken = state.accessToken
    if (state?.refreshToken) _externalRefreshToken = state.refreshToken
  },
})

// useAuth.ts — 不再直接操作 storage
setAuth(response.user, response.access_token, response.refresh_token)
```

### 变更 2：单一数据源

**Before:** Token 存在 `localStorage['access_token']` 和 `localStorage['auth-storage']` 两处

**After:** Token 只存在 `sessionStorage['auth-storage']` 一处，通过 `getAccessToken()` 供非 React 上下文读取

### 变更 3：水合感知

**Before:** `useAuthCheck` 直接读 `localStorage`（同步），与 Zustand 水合无关

**After:** `useAuthCheck` 读 Zustand store，使用 `persist.onFinishHydration` 等待水合完成

---

## 影响分析

### 不影响

- `LoginPage.tsx` — 通过 `useLogin` hook 间接操作
- `App.tsx` / `ProtectedRoute` — 通过 `useAuthCheck` hook 读取
- `AppLayout.tsx` — 通过 `useAuthStore` 读取
- `Sidebar.tsx` — 通过 props 接收 `userName`
- 后端 Rust 代码 — 完全无关

### 影响

- `apiClient.ts` — 请求拦截器改用 `getAccessToken()`，响应拦截器改用 `store.setTokens()`
- 测试文件 — 断言从 `localStorage.getItem()` 改为 `useAuthStore.getState()`

### 行为变更

| 场景 | Before | After |
|------|--------|-------|
| 关闭标签后重新打开 | 进入 Dashboard，显示「未登录」 | **进入登录页** ✅ |
| 新标签打开 | 进入 Dashboard，显示「未登录」 | **进入登录页** ✅ |
| 刷新页面 | 保持登录 | 保持登录 ✅ |

---

## 迁移路径

无需迁移。用户下次登录时 `sessionStorage` 会自动写入，旧的 `localStorage` 残留数据不会影响新逻辑（因为新代码不再读取 `localStorage`）。

如需清理用户浏览器中的旧 `localStorage` 数据，可在 `useAuthStore` 初始化时添加一行清理逻辑：

```typescript
// 可选：清理旧版 localStorage 残留
localStorage.removeItem('access_token')
localStorage.removeItem('refresh_token')
localStorage.removeItem('token_expires_at')
```

---

## 防止复发

1. **架构约束**: 认证状态只通过 `useAuthStore` 读写，在 CLAUDE.md 中注明
2. **静态分析**: ESLint 规则禁止 auth 模块直接使用 `localStorage`
3. **测试覆盖**: 新增「不写入 localStorage」测试用例作为回归防线
