# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260516-001 |
| **Title** | 聚合配置 Tab 缺少参数来源配置和必填状态调整 UI |
| **Type** | ui-ux |
| **Severity** | High |
| **Status** | open |
| **Analyzed At** | 2026-05-16 |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 001 |

---

## 1. Bug Description

### 1.1 Summary

在 Skill 详情页的「聚合配置」Tab 中，用户无法：
1. **指定参数来源** — 没有界面可以选择参数值的来源（`default` / `system_context` / `upstream_response` / `computed` / `user_input`）
2. **调整参数是否必填** — 对于来自接口的参数，仅显示静态文本「由接口定义」，无法切换必填/选填状态

### 1.2 Reproduction Steps

1. 导航到 `http://localhost:5174/skills/{skillId}`
2. 点击「聚合配置」Tab
3. 观察只有「参数覆盖配置」和「聚合参数 Schema」两个区块
4. 无法找到参数来源配置入口
5. 切换到「参数映射」Tab
6. 接口参数行只有「由接口定义」文本，没有必填切换开关

### 1.3 Expected Behavior

- 聚合配置 Tab 应包含参数来源（Param Source）配置区块，允许为每个参数指定值来源类型和配置
- 参数映射矩阵中的接口参数应支持必填/选填切换（与自定义参数一致）

### 1.4 Actual Behavior

- 聚合配置 Tab 只有 `ParamOverridePanel`（覆盖配置）和 `AggregateSchemaPreview`（Schema 预览）
- API 参数的必填状态完全由接口定义决定，用户无法覆盖

---

## 2. Code Graph Analysis

### 2.1 Affected Component Hierarchy

```
SkillDetailPage.tsx
├── AggregateConfigTab (line 322-351) ← 缺少 ParamSource 配置
│   ├── ParamOverridePanel.tsx         ✅ 已实现（覆盖配置）
│   └── AggregateSchemaPreview.tsx     ✅ 已实现（只读预览）
│       └── ParamSourceTag.tsx          ✅ 已实现（来源标签展示）
└── ParameterMappingMatrix.tsx (line 16-243)
    ├── renderCustomParams()            ✅ 自定义参数有必填 Switch
    └── renderApiParams()               ❌ 接口参数无必填 Switch（line 213: 仅显示"由接口定义"）
```

### 2.2 Backend API — 已完整实现

| API | Method | 路径 | 状态 |
|-----|--------|------|------|
| 参数来源查询 | GET | `/skills/{id}/param-sources` | ✅ |
| 参数来源保存 | POST | `/skills/{id}/param-sources` | ✅ |
| 必填状态更新 | PUT | `/skills/{id}/param-required` | ✅ |
| 参数配置查询 | GET | `/skills/{id}/parameter-config` | ✅ |

### 2.3 Frontend Hooks — 已完整实现

| Hook | 文件 | 状态 |
|------|------|------|
| `useParamSources(skillId)` | `useSkills.ts:247` | ✅ |
| `useSaveParamSources()` | `useSkills.ts:255` | ✅ |
| `useUpdateParamRequired()` | `useSkills.ts:188` | ✅ |

### 2.4 Frontend Types — 已完整实现

| Type | 文件 | 说明 |
|------|------|------|
| `ParamSourceType` | `types/index.ts:217` | `'default' \| 'system_context' \| 'upstream_response' \| 'computed' \| 'user_input'` |
| `SkillParamSource` | `types/index.ts:238` | 完整的参数来源模型 |
| `ParamSourceInput` | `types/index.ts:249` | 保存请求的输入类型 |
| `UpdateParamRequiredRequest` | `types/index.ts:119` | `param_source: 'custom' \| 'api'`，已支持 `api` |

### 2.5 数据流断裂点

```
Backend API (完整) → skillsApi.ts (完整) → useSkills.ts hooks (完整) → ❌ UI 组件层断裂
```

---

## 3. Root Cause Analysis

### 3.1 根因：UI 层实现不完整

后端 API、前端 API 客户端、hooks 和类型定义已全部实现（Story 3-7、Story 3-9），但 UI 组件层未完成对接：

**问题 1：`AggregateConfigTab` 缺少参数来源配置区块**

`SkillDetailPage.tsx:322-351` — `AggregateConfigTab` 组件只渲染了两个子组件：
- `ParamOverridePanel`（Story 3-5 覆盖配置）
- `AggregateSchemaPreview`（Story 3-6 Schema 预览）

缺少一个专门的参数来源配置组件来消费 `useParamSources` / `useSaveParamSources` hooks。

**问题 2：`ParameterMappingMatrix` 接口参数无必填切换**

`ParameterMappingMatrix.tsx:211-215` — 对于非排除的 API 参数，代码直接渲染了静态文本：
```tsx
{!param.is_excluded && (
  <span className="text-xs text-muted-foreground">
    由接口定义
  </span>
)}
```
没有提供 Switch 组件让用户覆盖必填状态。但后端 `UpdateParamRequiredRequest` 已支持 `param_source: 'api'`。

### 3.2 Bug 类型

**UI/UX** — 缺失的 UI 实现。后端完全就绪，前端数据层完全就绪，仅 UI 层未完成。

---

## 4. Fix Proposal

### 4.1 修复方案

**修复 1：在 `ParameterMappingMatrix` 中为 API 参数添加必填切换**

文件：`admin-frontend/src/features/skills/components/ParameterMappingMatrix.tsx`

将接口参数的「由接口定义」文本替换为 Switch + Tooltip（与自定义参数的 UI 模式一致），调用 `handleRequiredToggle` 但使用 `param_source: 'api'`。

需要修改点：
- `renderApiParams()` 中非排除参数的右侧操作区域（line 209-225）
- `handleRequiredToggle` 已支持自定义参数，需扩展支持 API 参数（使用 `param.source` 作为 `source_id`）

**修复 2：创建 `ParamSourceConfig` 组件**

文件：`admin-frontend/src/features/skills/components/ParamSourceConfig.tsx`（新建）

创建参数来源配置组件，功能：
- 读取聚合参数列表（从 `useParameterConfig`）
- 为每个参数提供来源类型选择（Select: default / system_context / upstream_response / computed / user_input）
- 根据来源类型动态渲染配置表单：
  - `default` → 默认值输入
  - `system_context` → 系统上下文字段路径选择
  - `upstream_response` → 上游接口 ID + 响应路径
  - `computed` → 计算代码编辑
  - `user_input` → 无额外配置
- 保存时调用 `useSaveParamSources`

**修复 3：在 `AggregateConfigTab` 中集成 `ParamSourceConfig`**

文件：`admin-frontend/src/features/skills/components/SkillDetailPage.tsx`

在 `AggregateConfigTab` 的「参数覆盖配置」和「聚合参数 Schema」之间插入新的「参数来源配置」区块。

### 4.2 涉及文件

| 文件 | 操作 | 预估行数 |
|------|------|----------|
| `ParameterMappingMatrix.tsx` | 修改 | ~30 行 |
| `ParamSourceConfig.tsx` | 新建 | ~200 行 |
| `SkillDetailPage.tsx` | 修改 | ~10 行 |

### 4.3 风险评估

- **低风险** — 仅添加 UI 层代码，后端 API 无需变更
- 所有 hooks 和 API 已存在且可用
- 不影响现有功能

### 4.4 验证计划

1. 在聚合配置 Tab 中，确认每个参数都有来源类型选择下拉框
2. 切换来源类型后，确认动态表单正确渲染
3. 保存参数来源后，确认 `AggregateSchemaPreview` 中的来源标签更新
4. 在参数映射 Tab 中，确认 API 参数旁有必填/选填切换开关
5. 切换必填状态后，确认 UI 状态正确更新（乐观更新 + 回滚）
