# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260525-007 |
| **Title** | 接口列表页测试弹窗缺少参数分组输入 |
| **Type** | ui-ux |
| **Severity** | High |
| **Status** | fixed |
| **Analyzed At** | 2026-05-25 |
| **Project** | diy-a2ui-v1.2 |

---

## 1. Bug Description

### 1.1 Summary

在接口列表页点击测试按钮打开 `TestAndInferDialog` 弹窗时，对于定义了 `param_groups`（参数分组）的接口，弹窗中没有参数分组的输入区域。用户只能看到一个 JSON 编辑器，不知道需要输入哪些分组字段，导致无法正确测试这类接口。

### 1.2 Reproduction Steps

1. 注册一个包含 `param_groups` 定义的 API 接口
2. 进入接口列表页
3. 点击该接口行的测试按钮（烧杯图标）
4. 观察弹窗：只有 JSON 编辑器，没有参数分组输入区域

### 1.3 Expected Behavior

弹窗应像测试代理页（`TestProxyPage`）一样，展示参数分组的 Tab 输入面板，让用户可以逐字段输入分组参数值。

### 1.4 Actual Behavior

弹窗只提取了 `params_schema.params`，完全忽略了 `params_schema.param_groups`，用户无从得知需要填写哪些分组字段。

---

## 2. Code Graph Analysis

### 2.1 Affected Code Paths

```
ApiList.tsx
  └─ handleTest() → setTestTarget(api)
       └─ TestAndInferDialog.tsx (弹窗)
            ├─ paramDefs = api.params_schema.params     ✅ 已提取
            ├─ paramGroups = api.params_schema.param_groups  ❌ 未提取
            ├─ handleRunTest()                           ❌ 未合并分组值
            └─ UI: 只有 JsonEditor                       ❌ 无分组输入
```

对比参照：`TestProxyPage.tsx`（测试代理页）完整实现了参数分组支持。

### 2.2 Key Files

| File | Role | Issue |
|------|------|-------|
| `admin-frontend/src/features/api-registrations/components/TestAndInferDialog.tsx` | 测试弹窗 | 缺少 paramGroups 提取、UI、编码逻辑 |
| `admin-frontend/src/features/test-proxy/components/TestProxyPage.tsx` | 测试代理页 | 正确实现，作为参照 |

---

## 3. Root Cause Analysis

### 3.1 Immediate Cause

`TestAndInferDialog` 在开发时只考虑了 `params` 字段，未同步实现对 `param_groups` 的支持。这是一个功能遗漏，而非逻辑错误。

### 3.2 Underlying Cause

`param_groups` 是后续迭代新增的功能（commit `010bdcf`），添加时只更新了 `TestProxyPage`，遗漏了 `TestAndInferDialog`。两个组件实现了相同的"API 测试"功能但没有共享逻辑，导致功能不同步。

---

## 4. Prevention Strategy

### 4.1 Workflow Change

当新增 `param_groups` 等跨组件功能时，应全局搜索所有消费 `params_schema` 的组件，确保同步更新。可通过 `codegraph_impact` 分析 `params_schema` 的所有消费者。

---

## 5. Fix Proposal

### 5.1 Fix Path

直接修改 `TestAndInferDialog.tsx`，参照 `TestProxyPage` 的实现模式添加参数分组支持。

### 5.2 Changes Made

文件：`admin-frontend/src/features/api-registrations/components/TestAndInferDialog.tsx`

1. **Imports**：新增 `Input`、`Tabs` 组件族、`Info` 图标、`ParamGroup` 类型
2. **State**：新增 `groupFieldValues` 存储分组字段输入值
3. **Memo**：新增 `paramGroups` 从 `params_schema.param_groups` 提取分组定义
4. **Logic**：`handleRunTest` 中将分组字段值按类型转换后合并到 `parameters`
5. **UI**：在 JSON 编辑器上方添加参数分组 Tab 输入面板

### 5.3 Verification

- TypeScript 编译零错误（`TestAndInferDialog.tsx` 无新增错误）
- UI 结构与 `TestProxyPage` 一致

---

## 6. Automated Verification Mechanism

### 6.1 Suggested Tests

- 对包含 `param_groups` 的 API 点击测试按钮，验证弹窗显示分组 Tab 输入面板
- 填入分组字段值后点击发送，验证参数正确传递到后端
- 对不含 `param_groups` 的 API 点击测试按钮，验证行为不变
