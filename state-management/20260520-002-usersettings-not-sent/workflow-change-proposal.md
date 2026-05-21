# Workflow Change Proposal — UserSettings Not Sent With Chat Request

**Bug ID**: 20260520-002-usersettings-not-sent
**日期**: 2026-05-20

---

## 当前问题

### 问题描述

`settings-store.initializeSettings()` 函数有完整定义和测试覆盖，但从未在组件树中被调用。页面刷新后，用户偏好仅依赖 localStorage 缓存恢复，新浏览器或清缓存场景下偏好丢失。

### 具体表现

- `ChatPage.tsx` 的 `useEffect` 只调用了 `initializeStore()`（chat-store），遗漏了 `initializeSettings()`（settings-store）
- 单元测试覆盖了 `initializeSettings` 的逻辑正确性，但未覆盖"函数是否在正确的生命周期被调用"
- 从 `credentials`（硬编码 Schema）迁移到 `preferences`（动态键值对）的重构过程中，初始化调用链断裂未被发现

---

## 提议变更

### 变更 1：组件初始化完整性检查

**Before:**
```typescript
// ChatPage.tsx — 只初始化一个 store
useEffect(() => {
    initializeStore();
}, [initializeStore]);
```

**After:**
```typescript
// ChatPage.tsx — 初始化所有相关 store
useEffect(() => {
    initializeStore();
}, [initializeStore]);

useEffect(() => {
    initializeSettings();
}, [initializeSettings]);
```

### 变更 2：集成测试覆盖初始化调用

**Before:** 无集成测试验证页面加载时的初始化行为

**After:** 新增集成测试断言 `ChatPage` 挂载时同时触发 `initializeStore()` 和 `initializeSettings()`

### 变更 3：统一 API 序列化行为

**Before:** `GET /api/settings` 返回未 JSON.parse 的原始字符串，`GET /api/preferences` 返回 JSON 解析后的值

**After:** `initializeSettings` 改用 `getPreferences()` API（`GET /api/preferences`，已正确 JSON.parse），避免序列化不一致

---

## 影响分析

### 不影响

- `chat-store.ts` — 已正确从 `settings-store` 读取 `preferences`
- `api-client.ts` — 已正确构建 `userSettings` 字段
- `PreferencesForm.tsx` — 保存逻辑不变
- 后端 — 无变更

### 影响

- `ChatPage.tsx` — 新增 `initializeSettings()` 调用（+2 行）
- `settings-store.ts` — `initializeSettings` 实现重构（改用 `getPreferences` + `Promise.allSettled`）
- 测试文件 — 需适配新的 API 调用模式

### 行为变更

| 场景 | Before | After |
|------|--------|-------|
| 新浏览器首次访问 | 偏好丢失，Skill 无法使用 | 从 API 加载偏好，Skill 正常工作 |
| 清缓存后刷新 | 偏好丢失 | 从 API 恢复偏好 |
| 正常刷新（localStorage 有缓存） | 正常 | 正常（优先用 API 数据） |

---

## 迁移路径

无需迁移。`initializeSettings()` 调用是纯前端变更，页面刷新即生效。API 端点 `GET /api/preferences` 已存在，无需后端变更。

---

## 防止复发

1. **架构约束**: 每个 store 的 `initialize*` 函数必须在组件入口调用，在 CLAUDE.md 中注明
2. **集成测试**: 新增页面挂载时的初始化调用验证
3. **重构检查清单**: 重构 store 时必须验证初始化调用链完整性
4. **命名约定**: store 初始化函数统一命名为 `initialize{StoreName}`，便于 grep 检查调用点
