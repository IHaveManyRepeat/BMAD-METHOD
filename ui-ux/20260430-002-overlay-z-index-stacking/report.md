# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | `20260430-002` |
| **Title** | 弹框蒙层(overlay)遮挡弹框内容(content) |
| **Type** | `ui-ux` — 界面/体验问题 |
| **Severity** | High |
| **Status** | fixed |
| **Analyzed At** | 2026-04-30 |
| **Project** | diy-a2ui-master |
| **Version** | v1.2 |

---

## 1. Bug Description

### 1.1 Summary

所有弹框（Dialog、Sheet）打开时，半透明蒙层渲染在弹框内容之上，导致整个屏幕被蒙层颜色覆盖，弹框内容不可见。

### 1.2 Reproduction Steps

1. 启动 admin-frontend 开发服务器
2. 导航到用户管理/环境管理/接口注册等任意页面
3. 点击"新增"或"编辑"按钮触发 Dialog/Sheet
4. 观察：屏幕变为半透明黑色蒙层，弹框内容被遮挡

### 1.3 Expected Behavior

蒙层（overlay）应渲染在弹框内容（content）之下，形成 `页面内容 < 蒙层 < 弹框内容` 的层级关系。

### 1.4 Actual Behavior

蒙层与弹框内容处于同一 z-index 层级（`z-50`），在某些渲染条件下蒙层覆盖了弹框内容。

---

## 2. Code Graph Analysis

### 2.1 Affected Code Paths

```
admin-frontend/src/
├── components/ui/
│   ├── dialog.tsx          ← 自定义 Dialog（蒙层 z-50 + 内容 z-50）
│   ├── sheet.tsx           ← 自定义 Sheet（蒙层 z-50 + 内容 z-50）
│   └── alert-dialog.tsx   ← Radix UI（蒙层 z-50 + 内容 z-50，但内部已处理）
├── shared/components/
│   └── GlobalSearch.tsx    ← 蒙层包裹内容（父子关系，无此问题）
└── index.css               ← Tailwind v4 入口（无 z-index token 定义）
```

### 2.2 Call Graph Hotspots

| Node | Connections | Complexity | Role |
|------|-------------|------------|------|
| `dialog.tsx:DialogContent` | 5（UserForm, ApiForm, EnvironmentForm 等） | 低 | Dialog 内容渲染，蒙层与内容 z-index 相同 |
| `sheet.tsx:SheetContent` | 3 | 低 | Sheet 内容渲染，蒙层与内容 z-index 相同 |
| `alert-dialog.tsx:AlertDialogContent` | 3（UserList, ResetPasswordDialog） | 低 | 基于 Radix UI，内部已正确分层 |

### 2.3 Dependency Chain

```
用户点击按钮 → DialogTrigger/SheetTrigger
  → DialogContent/SheetContent 渲染
    → Fragment(<>) 内同时渲染蒙层 div + 内容 div
      → 两者 z-index 均为 50 → 层级冲突
        → 蒙层可能覆盖内容
```

### 2.4 Visualization

**Code Graph HTML**: 未生成（无 bug-graph-generator.js 工具）

---

## 3. Root Cause Analysis

### 3.1 Immediate Cause

自定义 `DialogContent` 和 `SheetContent` 组件中，蒙层 `<div>` 和内容 `<div>` 使用了相同的 `z-50`（即 `z-index: 50`），作为同层级兄弟 `fixed` 元素，在 Tailwind CSS v4 的 `@layer` 级联机制下，渲染顺序不确定。

**关键代码**（`dialog.tsx:60-76`）：
```tsx
return (
  <>
    <div className="fixed inset-0 z-50 bg-black/50" />     {/* 蒙层: z-50 */}
    <div className="fixed left-1/2 top-1/2 z-50 ...">       {/* 内容: z-50 */}
      {children}
    </div>
  </>
)
```

### 3.2 Root Cause

**项目缺少 z-index 层级设计规范。** 所有 UI 组件（Dialog、Sheet、AlertDialog、Tooltip、Select）统一使用 `z-50`，开发者没有可遵循的层级约束，导致：

1. 不同语义的元素（蒙层 vs 内容 vs 下拉菜单 vs 浮动提示）共享同一 z-index
2. 没有 z-index token 体系，每次写新组件都是"复制粘贴 + 猜数字"
3. 无静态分析规则检测 z-index 冲突

### 3.3 Workflow-Level Cause

**为什么工作流允许这个 bug 发生？**

1. **UI 组件开发无层级规范** — 项目没有定义 z-index 使用标准，开发者在创建 UI 组件时随意使用 Tailwind 默认的 `z-50`
2. **代码审查未覆盖 z-index 层级** — PR review 关注功能实现，未检查 CSS 层叠上下文
3. **无自动化检测** — 没有 lint 规则或 CI 检查来发现 z-index 冲突

### 3.4 Similar Patterns

项目中所有使用 `z-50` 的组件都存在潜在风险：

| 组件 | z-index | 风险 |
|------|---------|------|
| Dialog 蒙层 | z-50 | ⚠️ 与内容冲突（已修复） |
| Dialog 内容 | z-50 | — |
| Sheet 蒙层 | z-50 | ⚠️ 与内容冲突（已修复） |
| Sheet 内容 | z-50 | — |
| AlertDialog 蒙层 | z-50 | ✅ Radix 内部处理 |
| AlertDialog 内容 | z-50 | ✅ Radix 内部处理 |
| Tooltip | z-50 | ⚠️ 可能与弹框冲突（已修复） |
| Select | z-50 | ⚠️ 可能与弹框冲突（已修复） |
| GlobalSearch 蒙层 | z-50 | ✅ 父子关系无冲突（已修复） |

---

## 4. Prevention Strategy

### 4.1 Workflow Change

| Current | Proposed |
|---------|----------|
| 开发者随意使用 `z-50`、`z-[999]` 等魔法数字 | 使用语义化 z-index token：`z-overlay`、`z-modal`、`z-dropdown` 等 |
| 无 z-index 层级规范 | 在 `index.css` 中通过 Tailwind v4 `@theme` 定义层级 token |
| 代码审查不检查 z-index | 审查清单增加"z-index 是否使用 token"检查项 |
| 无自动化检测 | 可通过 grep 脚本检测 `z-50`、`z-\[` 等违规用法 |

### 4.2 Automated Validation

已在 `index.css` 中建立 z-index 层级 token 体系：

```css
@theme {
  --z-base: 10;      /* 常规内容区 */
  --z-dropdown: 20;   /* 下拉菜单、Tooltip */
  --z-overlay: 30;    /* 蒙层/遮罩 */
  --z-modal: 40;      /* 弹框内容 */
  --z-toast: 50;      /* Toast 通知 */
}
```

---

## 5. Fix Proposal

### 5.1 Fix Path

**Decision**: `plan` — 直接修改，影响文件少（7 个文件），无需创建 story。

### 5.2 Validation Plan

See: [`./validation-plan.md`](./validation-plan.md)

### 5.3 Workflow Change Proposal

See: [`./workflow-change-proposal.md`](./workflow-change-proposal.md)

---

## 6. Automated Verification Mechanism

### 6.1 Unit Tests

- [ ] 无需单元测试 — 这是纯 CSS z-index 问题，不涉及逻辑

### 6.2 Integration Tests

- [ ] `e2e/dialog.spec.ts` — 验证 Dialog 打开后内容可见（蒙层未遮挡）
- [ ] `e2e/sheet.spec.ts` — 验证 Sheet 打开后内容可见
- [ ] `e2e/alert-dialog.spec.ts` — 验证 AlertDialog 打开后内容可见

### 6.3 Static Analysis

- [ ] 添加 CI 脚本：`grep -r "z-50\|z-40\|z-\[" admin-frontend/src/` 检测硬编码 z-index
- [ ] 可选：ESLint 自定义规则禁止 `z-{数字}` 以外的 z-index 用法

### 6.4 Runtime Checks

- [ ] 无需运行时检查 — z-index 是编译时确定的静态值

---

## 7. Impact Assessment

| Area | Impact |
|------|--------|
| Workflows affected | 无工作流影响 |
| Files modified | `dialog.tsx`, `sheet.tsx`, `alert-dialog.tsx`, `tooltip.tsx`, `select.tsx`, `GlobalSearch.tsx`, `index.css`（共 7 文件） |
| Tests required | 3 个 E2E 测试（建议） |
| Migration needed | 否 — 向后兼容 |

---

## 8. Lessons Learned

1. **Z-Index 必须建立 token 体系** — 没有语义化 token，z-index 就是"魔法数字"，迟早冲突
2. **蒙层与内容必须明确分层** — overlay(z-30) < modal(z-40)，不能依赖 DOM 顺序
3. **Radix UI 等成熟库已处理此类问题** — 自定义实现需自行处理层级关系
4. **CSS 层叠上下文是常见陷阱** — 代码审查应增加 CSS 层叠检查项

---

*Generated by `bmad-analyze-bug` workflow*
