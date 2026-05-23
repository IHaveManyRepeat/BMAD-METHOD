# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260523-001 |
| **Title** | 点击齿轮图标触发 4 次 `/api/preferences/schema` 请求 |
| **Type** | performance |
| **Severity** | MEDIUM |
| **Status** | fixed (code) + blocked (deployment) |
| **Analyzed At** | 2026-05-23T10:15:00.000Z |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 003 |

---

## 1. Bug Description

### 1.1 Summary

用户端点击齿轮（设置）图标后，浏览器 Network 面板显示向 `http://localhost:5173/api/preferences/schema` 发送了 **4 次** HTTP 请求。同样 `/api/preferences` 也会被请求 4 次。总计 8 个网络请求，其中绝大部分是冗余的。

经过两轮代码修复均未在运行时生效，最终发现 **真正的阻塞原因是 `src/` 目录中残留的 274 个旧编译产物（`.js`/`.d.ts`/`.map`），Vite 按默认解析优先级加载了这些旧文件，导致所有 `.tsx` 源码修改从未被执行。**

### 1.2 Reproduction Steps

1. 启动用户端前后端（`packages/frontend` + `packages/backend`）
2. 打开浏览器 DevTools → Network 面板
3. 点击页面右上角齿轮图标
4. 观察 Network 面板出现 4 次 `/api/preferences/schema` 请求

### 1.3 Expected Behavior

点击齿轮后仅发送 **1 次** `/api/preferences/schema` 请求。

### 1.4 Actual Behavior

发送 **4 次** `/api/preferences/schema` 请求（开发环境；生产环境为 2 次）。

---

## 2. Root Cause Analysis（三层根因）

### 2.1 表面原因：组件架构导致重复请求

| # | 因素 | 位置 | 影响倍数 |
|---|------|------|----------|
| 1 | **两个 `PreferencesForm` 实例** | `SettingsPanel.tsx:109,147` | ×2 |
| 2 | **React StrictMode 双重挂载** | `main.tsx:7` | ×2 |
| 3 | **无请求去重/缓存** | `preferencesApi.ts` | ×1 |

**计算**：2 个表单 × 2 次 StrictMode 挂载 = **4 次请求**

### 2.2 调用链

```
齿轮点击 → setIsOpen(true)
  → SettingsPanel 渲染两个 PreferencesForm:
    → PreferencesForm #1 (category="credentials")
      → useEffect mount → getPreferenceSchema() → GET /api/preferences/schema
      → StrictMode unmount + remount → useEffect mount → getPreferenceSchema() → GET /api/preferences/schema
    → PreferencesForm #2 (无 category)
      → useEffect mount → getPreferenceSchema() → GET /api/preferences/schema
      → StrictMode unmount + remount → useEffect mount → getPreferenceSchema() → GET /api/preferences/schema
总计: 4 × GET /api/preferences/schema + 4 × GET /api/preferences = 8 个请求
```

### 2.3 真正阻塞原因：旧编译产物劫持模块解析

**发现过程**：两轮代码修复（API 去重 + 组件级 Schema 提升）均未生效。重启服务器 + Ctrl+Shift+R 后仍为 4 次请求。

排查发现 `src/` 目录中存在 **274 个旧编译产物**（`.js`、`.d.ts`、`.js.map`、`.d.ts.map`），它们是由 `tsc` 编译生成的（`tsconfig.base.json` 设置了 `"declaration": true, "sourceMap": true`）。

**Vite 的模块解析顺序**（`resolve.extensions` 默认值）：

```
['.mjs', '.js', '.mts', '.ts', '.jsx', '.tsx', '.json']
          ^^^                                         ^^^
          第2位                                        第7位
```

`.js` 优先级高于 `.tsx`。当 `SettingsPanel.tsx` 执行 `import { PreferencesForm } from "./PreferencesForm"` 时：

```
Vite 解析 "./PreferencesForm":
  1. ./PreferencesForm.mjs  → 不存在
  2. ./PreferencesForm.js   → 存在！← 命中旧编译产物
  3. ./PreferencesForm.ts   → 跳过
  4. ./PreferencesForm.tsx  → 跳过（永远到不了这里）
```

**结果：所有对 `.tsx` 源文件的修改从未被 Vite 加载。运行时执行的始终是旧的 `.js` 文件。**

### 2.4 旧编译产物证据

**`src/chat/components/SettingsPanel.js`** — 旧版，无 schema state、无 useEffect：
```js
export function SettingsPanel() {
    const [isOpen, setIsOpen] = useState(false);
    // ... 没有 schema 相关逻辑
    // PreferencesForm 直接渲染，无 schema prop
    _jsx(PreferencesForm, { category: "credentials" })
    _jsx(PreferencesForm, {})
}
```

**`src/chat/api/preferencesApi.js`** — 旧版，无 Promise 去重：
```js
export async function getPreferenceSchema() {
    return apiClient("/preferences/schema");  // 无缓存，每次调用都发起请求
}
```

### 2.5 旧编译产物来源

`tsconfig.base.json` 中的配置：
```json
{
  "declaration": true,       // 生成 .d.ts
  "declarationMap": true,    // 生成 .d.ts.map
  "sourceMap": true          // 生成 .js.map
}
```

某次执行 `tsc`（非 `tsc --noEmit`）时，这些文件被输出到 `src/` 目录。由于 `packages/frontend/.gitignore` 未排除 `src/**/*.js` 等模式，它们未被忽略。

### 2.6 Bug 分类

| 层级 | 类型 | 说明 |
|------|------|------|
| 表面 | performance | 不必要的重复请求 |
| 表面 | state-management | 无缓存/去重 |
| 根源 | other（构建配置） | 旧编译产物劫持模块解析，代码修改无法生效 |

---

## 3. Fix Attempt 1: API 级别 Promise 去重（未成功）

### 3.1 方案描述

在 `preferencesApi.ts` 中添加模块级 Promise 缓存，使并发调用共享同一个 in-flight 请求。

### 3.2 代码变更

**文件**: `packages/frontend/src/chat/api/preferencesApi.ts`

```ts
let pendingSchemaPromise: Promise<ContextField[]> | null = null;
let pendingPreferencesPromise: Promise<Record<string, unknown>> | null = null;

export async function getPreferenceSchema(): Promise<ContextField[]> {
    if (!pendingSchemaPromise) {
        pendingSchemaPromise = apiClient<ContextField[]>("/preferences/schema").finally(() => {
            pendingSchemaPromise = null;
        });
    }
    return pendingSchemaPromise;
}

export async function getPreferences(): Promise<Record<string, unknown>> {
    if (!pendingPreferencesPromise) {
        pendingPreferencesPromise = apiClient<Record<string, unknown>>("/preferences").finally(() => {
            pendingPreferencesPromise = null;
        });
    }
    return pendingPreferencesPromise;
}
```

### 3.3 失败原因

**旧 `.js` 文件劫持了模块解析。** 修改写入了 `preferencesApi.ts`，但 Vite 加载的是 `preferencesApi.js`（旧版无去重逻辑）。`.tsx` 源文件的修改从未被执行。

### 3.4 教训

**修复未生效时，应首先确认运行时加载的是哪个文件。** 在 Vite 项目中，`src/` 目录内不应存在 `.js` 编译产物。

---

## 4. Fix Attempt 2: 组件级别 Schema 提升（代码正确，部署未生效）

### 4.1 方案描述

将 schema 获取逻辑从 `PreferencesForm` 提升到 `SettingsPanel`，由父组件统一获取一次，通过 props 下发给两个子表单。

### 4.2 架构变更

```
修复前:
  SettingsPanel
    ├── PreferencesForm #1 → useEffect → getPreferenceSchema() ← 独立请求
    └── PreferencesForm #2 → useEffect → getPreferenceSchema() ← 独立请求

修复后:
  SettingsPanel → useEffect → getPreferenceSchema() ← 唯一请求
    ├── PreferencesForm #1 ← schema prop → 仅 getPreferences()
    └── PreferencesForm #2 ← schema prop → 仅 getPreferences()
```

### 4.3 代码变更

#### 4.3.1 `SettingsPanel.tsx` — 新增 schema 获取/传递/重置

```tsx
const [schema, setSchema] = useState<ContextField[] | null>(null);

useEffect(() => {
    if (!isOpen || schema !== null) return;
    let cancelled = false;
    getPreferenceSchema()
        .then((result) => { if (!cancelled) setSchema(result); })
        .catch(() => { if (!cancelled) setSchema([]); });
    return () => { cancelled = true; };
}, [isOpen, schema]);

const handleClose = useCallback(() => {
    setIsOpen(false);
    setSchema(null);
}, []);

// JSX
{schema === null ? (
    <div className="...">加载用户凭证...</div>
) : (
    <PreferencesForm category="credentials" schema={schema} />
)}
```

#### 4.3.2 `PreferencesForm.tsx` — 新增 schema prop 条件分支

```tsx
interface PreferencesFormProps {
    readonly schema?: ContextField[];  // 新增
}

useEffect(() => {
    if (externalSchema !== undefined) {
        // 外部 schema 已提供 — 仅获取偏好值
        filteredFields = filterFieldsByCategory(externalSchema, category);
        savedPrefs = await getPreferences().catch(() => null);
    } else {
        // 无外部 schema — 原有行为
        const [schemaResult, preferencesResult] = await Promise.allSettled([
            getPreferenceSchema(),
            getPreferences(),
        ]);
    }
}, [category, externalSchema]);
```

### 4.4 失败原因

**与 Fix Attempt 1 相同：旧 `.js` 文件劫持了模块解析。** `SettingsPanel.js` 和 `PreferencesForm.js` 都是旧版编译产物，不包含任何修改。Vite 从未加载修改后的 `.tsx` 文件。

### 4.5 验证

- `tsc --noEmit` → 零错误
- `vitest run` → 338 测试全部通过
- **但测试通过是因为测试文件使用 `vi.mock` 替换了整个模块，绕过了模块解析问题。**

---

## 5. Fix Attempt 3: 清除旧编译产物（阻塞解除）

### 5.1 问题

`packages/frontend/src/` 目录中残留 274 个旧编译产物（`.js`、`.d.ts`、`.js.map`、`.d.ts.map`），它们由 `tsc` 生成（`tsconfig.base.json` 的 `declaration`、`sourceMap` 选项），但不应存在于 Vite 项目的源码目录中。

### 5.2 影响

Vite 默认 `resolve.extensions` 中 `.js` 优先于 `.tsx`，导致：
- 所有对 `.tsx` 源文件的修改**从未被运行时执行**
- 包括 Fix Attempt 1 和 Fix Attempt 2 的全部代码变更
- `tsc --noEmit` 和 `vitest` 不受影响（它们直接读取 `.tsx`）

### 5.3 修复步骤

1. **删除 `src/` 目录中所有编译产物**：

```bash
find packages/frontend/src \( -name "*.js" -o -name "*.d.ts" -o -name "*.js.map" -o -name "*.d.ts.map" \) -delete
```

2. **更新 `packages/frontend/.gitignore`**，防止未来重新生成：

```
# TypeScript 编译产物（Vite 项目由 esbuild 实时编译，不需要 tsc 输出）
src/**/*.js
src/**/*.d.ts
src/**/*.js.map
src/**/*.d.ts.map
```

3. **重启 Vite dev server + 硬刷新浏览器**

### 5.4 预期效果

Fix Attempt 1（API 去重）和 Fix Attempt 2（组件 Schema 提升）的代码将在删除旧 `.js` 文件后生效。`/api/preferences/schema` 请求从 4 次降为 1 次。

---

## 6. Fix Summary

| 指标 | 修复前 | Fix 1+2 生效后 |
|------|--------|----------------|
| `/api/preferences/schema` 请求数（开发） | 4 | **1** |
| `/api/preferences/schema` 请求数（生产） | 2 | **1** |
| `/api/preferences` 请求数（开发） | 4 | **2** |
| `/api/preferences` 请求数（生产） | 2 | **2** |

---

## 7. Lessons Learned

1. **Vite 项目中 `src/` 目录不应包含 `.js` 编译产物。** Vite 使用 esbuild 实时编译 `.tsx`/`.ts` 文件。`tsc` 输出的 `.js` 文件会因解析优先级劫持模块加载，导致源码修改无法生效。这是本 bug 中最关键的教训——两轮正确的代码修复都因这个问题而表现为"未修复"。

2. **`.gitignore` 应排除 Vite 项目源码目录中的编译产物。** 添加 `src/**/*.js`、`src/**/*.d.ts`、`src/**/*.js.map`、`src/**/*.d.ts.map`。

3. **`tsconfig.base.json` 的 `declaration` 和 `sourceMap` 选项适用于库项目，不适用于 Vite 应用项目。** 应用项目应在 `tsconfig.app.json` 中覆盖为 `"noEmit": true`，防止 `tsc` 输出文件到源码目录。

4. **测试通过 ≠ 修复生效。** 本案例中 `vitest` 全部通过，但运行时使用的是旧代码。因为测试通过 `vi.mock` 替换了模块，绕过了 Vite 的模块解析。验证修复是否生效必须检查浏览器运行时行为。

5. **组件架构层面的重复请求，应从组件层面根治。** 当多个子组件需要同一份数据时，获取逻辑应提升到共同父组件。API 级别的 Promise 缓存可作为防御性补充，但不能作为主方案。

---

*Bug analysis document updated. Status: code fixed, awaiting deployment fix (stale .js cleanup).*
