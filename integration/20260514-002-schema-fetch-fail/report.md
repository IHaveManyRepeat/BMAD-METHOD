# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260514-002 |
| **Title** | 页面没有调用接口去查找管理端设置的系统上下文字段 |
| **Type** | integration |
| **Severity** | High |
| **Status** | open |
| **Analyzed At** | 2026-05-14T15:05:00.000Z |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 002 |

---

## 1. Bug Description

### 1.1 Summary

用户访问 `http://localhost:5173/` 页面时，偏好设置面板中的表单完全没有调用 `/api/preferences/schema` 接口，导致表单显示为空。

**用户澄清**：不是"调用接口失败"，而是"根本没有发起调用"。

### 1.2 Reproduction Steps

1. 启动 backend 服务（`localhost:3000`）
2. 启动 frontend 开发服务器（`localhost:5173`）
3. 打开浏览器 DevTools → Network 面板
4. 访问 `http://localhost:5173/`
5. 点击"设置"按钮打开偏好设置面板
6. 观察 Network 面板 — **没有** `/api/preferences/schema` 请求发出

**预期行为**：打开设置面板时，自动调用 `GET /api/preferences/schema` 获取字段定义

**实际行为**：Network 面板中没有任何相关请求

### 1.3 Expected Behavior

`PreferencesForm` 组件在 mount 时（`useEffect`）应发起并行请求：

```typescript
const [schemaResult, preferencesResult] = await Promise.allSettled([
    getPreferenceSchema(),
    getPreferences(),
]);
```

### 1.4 Actual Behavior

接口调用未发生。

---

## 2. Code Graph Analysis

### 2.1 Call Chain

```
ChatSidebar.tsx
  └─ <SettingsPanel />

SettingsPanel.tsx (line 109, 147)
  └─ <PreferencesForm category="credentials" />
  └─ <PreferencesForm />

PreferencesForm.tsx (TSX source)
  └─ useEffect(() => { load(); }, [category])
       └─ Promise.allSettled([getPreferenceSchema(), getPreferences()])

preferencesApi.ts (TSX source)
  └─ getPreferenceSchema()
       └─ apiClient("/preferences/schema")

backend preferences.ts
  └─ GET /api/preferences/schema
       └─ fetchSchemaFromAdmin() → admin API
```

### 2.2 Key Discovery — Stale Build Artifact

检查 `PreferencesForm` 的 **编译产物**（`.js`）与 **源码**（`.tsx`）对比：

| 差异 | TSX 源码 | 编译后 JS |
|------|---------|----------|
| `category` prop | `export function PreferencesForm({ onSaved, category }: PreferencesFormProps)` | `export function PreferencesForm({ onSaved })` — **category 丢失** |
| `CREDENTIAL_FIELD_NAMES` | 存在，定义凭证字段白名单 | 编译产物中 **无此常量** |
| `filterFieldsByCategory` | 存在，按 category 过滤字段 | 编译产物中 **无此函数** |

**结论**：`PreferencesForm.js` 是 **旧版本编译产物**，不包含 `category` 相关的代码。Dev server 可能使用了过时构建。

### 2.3 Files Involved

| File | Role |
|------|------|
| `packages/frontend/src/chat/components/SettingsPanel.tsx` | 引用 `PreferencesForm`，传递 `category` prop |
| `packages/frontend/src/chat/components/PreferencesForm.tsx` | 源码，包含 `useEffect` 加载逻辑 |
| `packages/frontend/src/chat/components/PreferencesForm.js` | **编译产物（疑似过时）** |
| `packages/frontend/src/chat/api/preferencesApi.ts` | API 调用封装 |
| `packages/frontend/vite.config.ts` | Dev server 代理配置 |

---

## 3. Root Cause Analysis

### 3.1 Immediate Cause

`PreferencesForm` 的编译产物 `.js` 文件与 `.tsx` 源码不一致 — **编译产物缺少 `category` prop 和 `useEffect` 加载逻辑**。这导致 dev server 加载的是旧版组件，该版本可能没有自动发起 API 调用。

### 3.2 Root Cause

**构建产物与源码不同步**：`.tsx` 源码已更新（新增 `category` 支持、完整的 `useEffect` 加载），但 `.js` 编译产物未重新生成，或 dev server 缓存了旧版本。

### 3.3 Workflow-Level Cause

**为什么工作流允许这个 bug 发生？**

1. **前端无增量构建验证** — TSX 修改后未强制重新编译
2. **Dev server 缓存问题** — Vite 的 HMR 可能在某些情况下加载了过期的模块
3. **没有监控构建产物时效性** — CI/CD 未检查 `.js` 与 `.tsx` 的 timestamp 一致性

---

## 4. Prevention Strategy

### 4.1 Workflow Change

| Current | Proposed |
|---------|----------|
| Dev server 直接从 `.tsx` 源码 HMR | 添加构建产物 timestamp 检查，不一致时警告或强制重 build |
| 前端 build 仅检查 tsc 错误 | 添加 `tsc --noEmit` 作为 pre-commit hook |

### 4.2 Automated Validation

1. **E2E 测试**：验证打开设置面板后 Network 有 `/api/preferences/schema` 请求
2. **构建检查**：比较 `.js` 与 `.tsx` timestamp，不一致时 CI 失败

---

## 5. Fix Proposal

### 5.1 Fix Path

**Decision**: `plan` — 重新构建前端，清理缓存即可解决，无需跨模块改动

### 5.2 Concrete Fix

1. **清理 build 缓存，重新编译**：
   ```bash
   rm -rf packages/frontend/src/chat/components/*.js*.map
   cd packages/frontend && npm run build
   ```

2. **重启 dev server** 确保加载最新模块

3. **验证**：
   - 打开 DevTools Network
   - 打开设置面板
   - 确认 `/api/preferences/schema` 请求发出

---

## 6. Automated Verification Mechanism

### 6.1 E2E Tests

- [ ] 打开设置面板 → 确认 Network 出现 `/api/preferences/schema` 请求
- [ ] 刷新页面 → 确认请求不重复发起（仅 panel 打开时一次）

### 6.2 Build Verification

- [ ] `npm run build` 无错误
- [ ] `.js` 文件 timestamp > 对应 `.tsx` 文件

---

*Bug Analysis Report updated with corrected root cause.*