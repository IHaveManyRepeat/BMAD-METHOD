# Bug Analysis: 接口测试页选择环境后未过滤接口列表

- **Date**: 2026-05-25
- **Severity**: Medium
- **Bug Type**: missing-validation
- **Status**: Fixed

## 1. Bug Description

### 1.1 Summary

接口测试页（TestProxyPage）选择环境后，接口下拉框仍显示所有环境的全部 API，而非仅展示该环境下的接口。

### 1.2 Reproduction

1. 进入管理端「接口测试」页面
2. 在环境选择器中选择某个具体环境
3. 观察接口选择器 → 显示了所有环境的 API（错误）
4. 预期：只显示 `environment_id` 匹配所选环境的 API

### 1.3 Affected Area

- **Frontend**: `admin-frontend/src/features/test-proxy/components/TestProxyPage.tsx`
- **Backend**: 无需改动（`ListApisQuery` 已支持 `environment_id` 过滤参数）

## 2. Code Graph Analysis

### 2.1 Affected Code Paths

```
TestProxyPage.tsx
  ├── useApiRegistrations()          → 加载全部 API，无环境过滤参数
  ├── apis?.map(...)                 → 接口下拉框直接遍历全部数据（第 280 行）
  └── setSelectedEnvId(v)            → 环境切换时未重置 selectedApiId
```

### 2.2 Backend Support (Already Working)

- `admin-backend/src/features/api/model.rs:ListApisQuery` — 已有 `environment_id: Option<Uuid>` 过滤字段
- `admin-backend/src/features/api/service.rs:list_apis()` — 已实现 SQL WHERE 条件过滤
- 前端 `apiRegistrationApi.list()` 未使用此参数

## 3. Root Cause Analysis

### 3.1 Immediate Cause

`TestProxyPage.tsx` 第 77 行调用 `useApiRegistrations()` 获取全部 API 后，第 280 行直接 `apis?.map(...)` 渲染到接口下拉框，未按 `selectedEnvId` 过滤。

### 3.2 Design Gap

1. **缺少客户端过滤逻辑**：加载了全量数据但没有按环境筛选
2. **状态联动缺失**：环境切换时未重置 API 选择状态，可能导致已选 API 不属于新环境

### 3.3 Why Bug Occurred

功能开发时接口测试页一次性加载全量 API 列表，未考虑环境与 API 的归属关系。后端已具备过滤能力但前端未对接。

## 4. Prevention Strategy

### 4.1 Development Workflow

- 新增「选择器联动」场景的检查项：当存在「A 选择器影响 B 选择器数据范围」时，必须实现过滤 + 状态重置
- 代码审查时关注 `useQuery` 返回数据是否需要二次过滤

### 4.2 Pattern Enforcement

建议抽取通用的「级联选择器」模式（Cascading Select），封装环境-接口联动逻辑，避免各页面重复实现。

## 5. Fix Proposal

### 5.1 Changes Made

**File**: `admin-frontend/src/features/test-proxy/components/TestProxyPage.tsx`

**Change 1** — 新增 `filteredApis` 过滤（第 93-99 行）：

```typescript
const filteredApis = useMemo(
  () =>
    selectedEnvId === '__placeholder__'
      ? apis
      : apis?.filter((a) => a.environment_id === selectedEnvId),
  [apis, selectedEnvId]
)
```

**Change 2** — 环境切换时重置 API 选择（第 270 行）：

```typescript
onValueChange={(v) => { setSelectedEnvId(v); setSelectedApiId('__placeholder__') }}
```

**Change 3** — 接口下拉框使用过滤后列表（第 288 行）：

```typescript
// apis?.map(...) → filteredApis?.map(...)
```

### 5.2 Impact

- 修改 1 个文件，3 处变更
- 无 API 契约变更
- 无破坏性变更：未选择环境时仍显示全部 API（向后兼容）

## 6. Automated Verification Mechanism

### 6.1 Recommended Tests

```typescript
// Test: 选择环境后过滤 API 列表
test('filters apis by selected environment', () => {
  render(<TestProxyPage />)
  // select environment
  // verify api dropdown only shows matching environment_id apis
})

// Test: 切换环境后重置 API 选择
test('resets api selection when environment changes', () => {
  render(<TestProxyPage />)
  // select env A, select api X
  // change to env B
  // verify api selection is reset to placeholder
})
```

### 6.2 Manual Verification

1. 进入接口测试页
2. 选择环境 → 确认接口下拉框仅显示该环境的 API
3. 选择一个 API
4. 切换到另一个环境 → 确认 API 选择被重置
5. 不选择环境时 → 确认显示全部 API
