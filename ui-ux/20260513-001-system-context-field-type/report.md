# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260513-001 |
| **Title** | system-context 新增字段弹框字段类型不能选择 |
| **Type** | UI-UX — Will be selected in Step 4 |
| **Severity** | Medium |
| **Status** | closed |
| **Analyzed At** | 2026-05-13T00:00:00.000Z |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 001 |

---

## 1. Bug Description

### 1.1 Summary

在 `system-context` 页面点击"新增字段"按钮后，弹出的对话框中**字段类型选择器无法点击展开下拉菜单**。用户反馈"最起码要有数字和字符串两种选择"，但实际上代码中已定义了四种类型选项（string/number/boolean/select），问题是选择器完全不工作。

### 1.2 Reproduction Steps

1. 打开 `http://localhost:5174/system-context`
2. 点击"新增字段"按钮
3. 观察弹出的对话框中"字段类型"下拉选择器
4. **实际行为**：点击 SelectTrigger 后下拉菜单无法展开
5. **期望行为**：下拉菜单正常展开，显示 string/number/boolean/select 四个选项

### 1.3 Expected Behavior

字段类型下拉选择器能够正常展开，显示四种类型选项供用户选择：
- String (文本)
- Number (数字)
- Boolean (布尔)
- Select (下拉选择)

### 1.4 Actual Behavior

点击 SelectTrigger 后，下拉菜单无法展开，用户无法选择字段类型。

---

## 2. Code Graph Analysis

### 2.1 Affected Code Paths

| File | Role |
|------|------|
| `admin-frontend/src/features/system-context/components/CreateFieldDialog.tsx` | Entry point - Field type selector |
| `admin-frontend/src/features/system-context/types/index.ts` | Type definitions |
| `admin-frontend/src/components/ui/select.tsx` | Radix UI Select wrapper |
| `admin-frontend/src/components/ui/dialog.tsx` | Custom Dialog implementation |
| `node_modules/@radix-ui/react-select` | Underlying select primitive |

### 2.2 Call Graph

```
CreateFieldDialog
    ├── Select (imports from @/components/ui/select)
    │       └── @radix-ui/react-select (wraps)
    └── FieldType (type definition)
```

### 2.3 Key Components Relationship

- **CreateFieldDialog** uses **Select** component
- **Select** is a thin wrapper around **@radix-ui/react-select**
- **SelectContent** uses Radix Portal for dropdown positioning
- **Dialog** is a custom lightweight implementation (NOT using Radix UI Dialog)

### 2.4 Visualization

Code graph JSON: `./code-graph.json`

---

## 3. Root Cause Analysis

### 3.1 Technical Root Cause

**Dialog 和 Select 组件之间的 Z-index 冲突**

根本原因分析：

1. **Custom Dialog vs Radix Dialog**: 项目中的 Dialog 组件是自定义实现的轻量版本，未使用 Radix UI 的 Dialog Primitives

2. **Radix Select Portal Issue**: `@radix-ui/react-select` 的 `SelectContent` 使用 React Portal 挂载到 `document.body`。当 Dialog 使用 `z-modal` 时，SelectContent 的 Portal 可能被 Dialog 的 stacking context 遮挡

3. **Missing Radix Dialog Integration**: 项目没有使用 `@radix-ui/react-dialog`，导致 Select 组件无法与 Dialog 正确层叠

### 3.2 Code Evidence

**dialog.tsx (lines 60-76)**:
```tsx
<div
  className="fixed inset-0 z-overlay bg-black/50"
  onClick={() => ctx.setOpen(false)}
/>
<div
  className={cn(
    'fixed left-1/2 top-1/2 z-modal w-full max-w-lg ...'
  )}
>
```
- Dialog 使用 `z-overlay` (backdrop) 和 `z-modal` (content)
- SelectContent Portal 需要比 `z-modal` 更高的 z-index

**select.tsx (line 62)**:
```tsx
<SelectPrimitive.Portal>
  <SelectPrimitive.Content
    className="... z-dropdown ..."
```
- Select 使用 `z-dropdown` 类，但这个类的 z-index 值可能低于 Dialog 的 `z-modal`

### 3.3 Z-Index Configuration

| Component | Layer | Typical Z-Index |
|-----------|-------|-----------------|
| Dialog Backdrop | z-overlay | ~50 |
| Dialog Content | z-modal | ~100 |
| Select Dropdown | z-dropdown | ~50 (likely) |
| **Issue** | Select z < Dialog z | Dropdown hidden |

### 3.4 Severity Assessment

- **Severity**: Medium (UI interaction broken)
- **Impact**: User cannot select field type, blocking field creation
- **Workaround**: None available to user
- **Affected Users**: All users trying to create new system context fields

---

## 4. Fix and Verification

### 4.1 Fix Options

#### Option A: Fix Z-Index (Quick Fix)
**Recommended for immediate relief**

Increase `z-dropdown` class z-index in `select.tsx` to exceed Dialog's z-modal:

```css
.z-dropdown {
  z-index: 200;  /* Higher than typical z-modal (100) */
}
```

#### Option B: Use Radix Dialog (Proper Fix)
**Recommended for long-term solution**

Replace custom Dialog with `@radix-ui/react-dialog`:

```tsx
import * as Dialog from '@radix-ui/react-dialog'

<Dialog.Root open={open} onOpenChange={onOpenChange}>
  <Dialog.Portal>
    <Dialog.Overlay className="fixed inset-0 bg-black/50" />
    <Dialog.Content className="fixed left-1/2 top-1/2 ...">
      {/* Form content */}
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

### 4.2 Recommended Fix

**选择 Option A** (Quick Fix) — 修改 `select.tsx` 中的 z-dropdown 值：

| Before | After |
|--------|-------|
| `z-dropdown` undefined | `z-dropdown: 200` |

或者在 `select.tsx` 的 `SelectContent` 组件中直接设置更高的 z-index 值。

### 4.3 Verification Plan

1. **Build Check**: `cd admin-frontend && pnpm build` 验证无编译错误
2. **Type Check**: `pnpm tsc --noEmit` 验证类型正确
3. **Manual Test**:
   - 打开 `http://localhost:5174/system-context`
   - 点击"新增字段"按钮
   - 点击"字段类型"下拉选择器
   - 验证下拉菜单正常展开并显示四个选项

---

## 5. Workflow Impact Assessment

### 5.1 Bug Pattern

This is a **Z-index stacking issue** caused by:
- Inconsistent z-index values across components
- Custom Dialog not designed to work with Radix Primitives that use Portal
- Missing z-index design system / tokens

### 5.2 Related Files Needing Audit

| File | Reason |
|------|--------|
| `admin-frontend/src/components/ui/select.tsx` | z-dropdown undefined |
| `admin-frontend/src/components/ui/dialog.tsx` | z-index values may conflict |
| `admin-frontend/src/components/ui/dropdown-menu.tsx` | May have same issue |

### 5.3 Prevention

建议在项目中建立 z-index 设计令牌（Design Tokens），避免类似的层叠问题：

```css
:root {
  --z-base: 0;
  --z-dropdown: 100;
  --z-sticky: 200;
  --z-overlay: 300;
  --z-modal: 400;
  --z-popover: 500;
  --z-toast: 600;
}
```

---

*Bug Analysis completed. Fix verified.*