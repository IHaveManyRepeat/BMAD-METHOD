# Validation Plan — Session Persist After Tab Close

**Bug ID**: 20260427-001-session-persist-after-tab-close
**日期**: 2026-04-27

---

## 自动化验证机制

### 1. 单元测试（已实施）

| 测试 | 验证内容 | 文件 |
|------|----------|------|
| `useAuthStore > persist 使用 sessionStorage 而非 localStorage` | 存储类型正确 | `useAuth.test.tsx` |
| `useAuthStore > setAuth 更新 store 状态和外部 getter` | 数据源一致性 | `useAuth.test.tsx` |
| `useAuthStore > logout 清除 store 状态和外部 getter` | 清理完整性 | `useAuth.test.tsx` |
| `useAuthStore > setTokens 更新 token 但保留 user` | Token 刷新不影响 user | `useAuth.test.tsx` |
| `useLogin > 登录后不写入 localStorage` | 杜绝跨会话持久化 | `useAuth.test.tsx` |
| `useLogout > 清除 store 认证状态、外部 getter` | 完全清理 | `useAuth.test.tsx` |
| `useAuthCheck > store 无/有/过期/无效 token` | 各种认证状态 | `useAuth.test.tsx` |
| `LoginPage > 登录成功后不写入 localStorage` | UI 层面验证 | `LoginPage.test.tsx` |

### 2. 静态分析规则（建议实施）

在 ESLint 配置中添加 `no-restricted-syntax` 规则，禁止在 `auth/` 模块中直接使用 `localStorage`：

```javascript
// .eslintrc — admin-frontend
{
  "rules": {
    "no-restricted-syntax": [
      "error",
      {
        "selector": "CallExpression[callee.object.name='localStorage'][callee.property.name='setItem']",
        "message": "auth 模块禁止直接使用 localStorage。请使用 useAuthStore 统一管理认证状态。"
      }
    ]
  },
  "overrides": [{
    "files": ["src/features/auth/**/*.{ts,tsx}"],
    "rules": {
      "no-restricted-syntax": [
        "error",
        {
          "selector": "CallExpression[callee.object.name='localStorage']",
          "message": "auth 模块禁止直接使用 localStorage。请使用 useAuthStore 统一管理认证状态。"
        }
      ]
    }
  }]
}
```

### 3. E2E 测试（建议实施）

| 测试场景 | 步骤 | 预期 |
|----------|------|------|
| 关闭标签后重新打开 | 登录 → 关闭标签 → 重新打开 URL | 进入登录页 |
| 新标签打开 | 登录 → 新标签打开同 URL | 进入登录页 |
| 正常使用中刷新 | 登录 → 刷新页面 | 保持登录状态 |

### 4. 架构约束

在 ADR 或 CLAUDE.md 中注明：
- 认证状态只通过 `useAuthStore` 读写
- 禁止直接操作 `localStorage` / `sessionStorage`
- Token 刷新通过 `store.setTokens()` 统一更新
