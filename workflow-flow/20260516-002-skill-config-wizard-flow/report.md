# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260516-002 |
| **Title** | 技能配置流程缺少步骤化向导引导，用户可跳过关键步骤直接启用 |
| **Type** | workflow-flow + ui-ux |
| **Severity** | HIGH |
| **Status** | open |
| **Analyzed At** | 2026-05-16 |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 002 |

---

## 1. Bug Description

### 1.1 Summary

Skill 详情页 (`SkillDetailPage`) 当前使用 `Tabs` 组件将 6 个配置步骤（依赖接口、参数映射、参数分析、聚合配置、骨架生成、提示词建议）呈现为并列标签页。用户可以任意跳转，无需按顺序完成。部分编辑操作通过 `Dialog` 弹框实现而非内联表单。启用验证仅检查依赖 API 是否 active，不验证中间配置步骤是否完成。各步骤无完成状态标识。

### 1.2 Reproduction Steps

1. 创建一个新 Skill
2. 进入 Skill 详情页
3. 不配置任何依赖接口，直接点击「参数映射」标签 — 无报错，显示空内容
4. 不配置参数，直接点击「启用」— 仅当有依赖 API 且 inactive 时才失败；若无依赖 API，直接启用成功
5. 启用后的 Skill 没有参数映射、聚合配置，在实际使用时可能出错

### 1.3 Expected Behavior

- 6 个配置步骤应呈现为**步骤化向导 (Stepper)**，每步依赖前一步完成
- 配置操作应是**内联表单**而非弹框
- 点击前面的步骤可以回退编辑
- 未完成所有必要步骤时，**启用按钮应被阻止**
- 启用时**提示用户具体缺少哪些步骤**
- 每个步骤有**完成状态标识**（✅ / ⚠️ / 🔴），引导用户知道还差哪些

### 1.4 Actual Behavior

- 6 个标签页可任意跳转，无顺序约束
- 依赖接口选择、自定义参数添加使用 `Dialog` 弹框
- 启用验证仅检查 `dependency_apis_active`，不检查配置完整性
- 无步骤完成状态标识

---

## 2. Code Graph Analysis

### 2.1 Affected Components (Frontend)

```
SkillDetailPage.tsx (主页面，当前使用 Tabs)
├── SkillApiDependencies (依赖接口，使用 Dialog)
│   └── SkillDependencySelector (弹框内选择器)
├── CustomParamsSection (自定义参数，使用 Dialog)
│   └── CustomParamDialog
├── ParameterMappingMatrix (参数映射矩阵)
├── ParameterAnalysisPanel (参数分析)
├── AggregateConfigTab (聚合配置)
│   ├── ParamOverridePanel (参数覆盖)
│   ├── ParamSourceConfig (参数来源)
│   └── AggregateSchemaPreview (聚合 Schema)
├── SkeletonSection (骨架生成)
├── PromptSuggestion (提示词建议)
└── LifecycleControls (启用/禁用按钮)
```

### 2.2 Affected API Endpoints (Backend)

```
POST /api/skills/:id/enable → enable_skill()
  └── repo::check_dependency_apis_active() — 仅检查依赖 API 状态
      ⚠️ 缺少：参数映射完整性检查
      ⚠️ 缺少：聚合配置完整性检查
```

### 2.3 Data Flow Dependencies

```
[依赖接口] ─→ [参数映射] ─→ [参数分析] ─→ [聚合配置] ─→ [骨架生成] ─→ [提示词建议]
     │              │              │              │              │
     │              │              │              │              └── 依赖骨架数据
     │              │              │              └── 依赖分析结果（冲突参数需覆盖）
     │              │              └── 依赖参数列表（分析冲突/共享）
     │              └── 依赖 API 参数（从 API schema 提取）
     └── 基础：定义 API 依赖关系
```

### 2.4 Key Files

| File | Role |
|------|------|
| `admin-frontend/src/features/skills/components/SkillDetailPage.tsx:362-581` | 主页面，Tabs 布局 |
| `admin-frontend/src/features/skills/components/SkillApiDependencies (inline):57-134` | 依赖接口，Dialog 模式 |
| `admin-frontend/src/features/skills/components/CustomParamsSection (inline):290-316` | 自定义参数，Dialog 模式 |
| `admin-frontend/src/features/skills/components/ParameterAnalysisPanel.tsx` | 参数分析面板 |
| `admin-frontend/src/features/skills/components/AggregateConfigTab (inline):323-360` | 聚合配置面板 |
| `admin-frontend/src/features/skills/components/SkeletonSection (inline):141-214` | 骨架生成 |
| `admin-frontend/src/features/skills/components/PromptSuggestion.tsx` | 提示词建议 |
| `admin-backend/src/features/skill/service.rs:181-219` | enable_skill 后端验证 |
| `admin-frontend/src/features/skills/types/index.ts` | 类型定义 |

---

## 3. Root Cause Analysis

### 3.1 Workflow-Level Causes

**W-1: 缺少步骤间依赖约束**
- Tabs 组件天然支持任意跳转，不适合有依赖关系的配置流程
- 应改用 Stepper/Wizard 模式，每步激活条件依赖前一步完成

**W-2: 启用验证不充分**
- `enable_skill()` 仅验证 `dependency_apis_active`，不验证配置完整性
- 应新增配置完整性检查：至少有依赖接口 → 无未解决的参数冲突 → 聚合配置有效

**W-3: 编辑模式不一致**
- 依赖接口和自定义参数使用 Dialog 弹框，而其他步骤是内联展示
- 用户感知割裂：有的需要弹框操作，有的直接内联操作

### 3.2 Implementation-Level Causes

**I-1: SkillDetailPage 无步骤状态管理**
- `SkillDetailPage` 组件中没有跟踪各步骤完成状态的 state
- 缺少 `stepStatus` / `stepCompletion` 之类的状态对象

**I-2: LifecycleControls 无配置完整性感知**
- `LifecycleControls` 组件仅接收 `skillId` 和 `currentStatus`
- 不了解各配置步骤的完成情况

**I-3: 无步骤完成判定逻辑**
- 每个步骤应该有一个明确的"完成"判定条件，目前没有定义：
  - 步骤1 完成 = 至少配置了 1 个依赖 API
  - 步骤2 完成 = 参数映射已查看/确认（自动）
  - 步骤3 完成 = 参数分析已执行
  - 步骤4 完成 = 所有冲突参数已处理 + 来源已配置
  - 步骤5 完成 = 骨架已生成（可选）
  - 步骤6 完成 = 提示词已查看（可选）

---

## 4. Fix Proposal

### 4.1 UI Redesign: Tabs → Stepper Wizard

将 6 个 Tabs 改造为 **Stepper 向导组件**：

```
┌─────────────────────────────────────────────────────────────┐
│  ✅ 1.依赖接口  →  ⚠️ 2.参数映射  →  ⭕ 3.参数分析  →  ...  │
└─────────────────────────────────────────────────────────────┘
│                                                              │
│  [当前步骤的内容区域 — 内联表单]                               │
│                                                              │
│  [上一步]                              [下一步/完成此步骤]     │
└──────────────────────────────────────────────────────────────┘
```

**状态标识：**
- ✅ 绿色勾 — 步骤已完成
- ⚠️ 黄色感叹号 — 步骤部分完成或有警告
- ⭕ 灰色圆圈 — 步骤未开始
- 🔒 锁定 — 前置步骤未完成，不可操作

**点击回退：** 已完成的步骤可点击回退编辑，编辑后不影响后续步骤的状态（除非数据变化导致后续依赖失效）。

### 4.2 Dialog → Inline Form

将以下 Dialog 改为内联表单：
- `SkillDependencySelector` — 从 Dialog 改为 Stepper 步骤1的内联内容
- `CustomParamDialog` — 从 Dialog 改为步骤2的内联子区域

### 4.3 Step Completion Logic

```typescript
type StepStatus = 'locked' | 'incomplete' | 'complete' | 'warning'

interface StepDefinition {
  id: string
  label: string
  status: StepStatus
  required: boolean        // 是否必须完成才能启用
  completionCheck: () => boolean | Promise<boolean>
}

// 步骤完成条件定义
const STEPS: StepDefinition[] = [
  { id: 'dependencies', label: '依赖接口', required: true,
    completionCheck: () => deps.length > 0 },
  { id: 'paramMapping', label: '参数映射', required: true,
    completionCheck: () => true }, // 自动完成，查看即算完成
  { id: 'paramAnalysis', label: '参数分析', required: true,
    completionCheck: () => analysisData !== null },
  { id: 'aggregateConfig', label: '聚合配置', required: true,
    completionCheck: () => conflictingParams.length === 0 || overrides.length > 0 },
  { id: 'skeleton', label: '骨架生成', required: false,
    completionCheck: () => skeleton !== null },
  { id: 'prompt', label: '提示词建议', required: false,
    completionCheck: () => promptData !== null },
]
```

### 4.4 Enhanced Enable Validation

**前端：** 在 `LifecycleControls` 中，启用按钮点击时：
1. 收集所有 required 步骤的完成状态
2. 如有未完成步骤，展示具体缺失列表而非直接调用 API
3. 用户确认后再调用后端 API

**后端：** 在 `enable_skill()` 中扩展验证：
- 当前：仅检查 `dependency_apis_active`
- 新增：检查是否存在依赖 API（至少 1 个）
- 可选：检查参数配置完整性

### 4.5 Implementation Plan

| Phase | Scope | Est. Lines |
|-------|-------|-----------|
| Phase 1 | 新建 `SkillConfigWizard` 组件，替代 Tabs | ~150 行 |
| Phase 2 | 新建 `StepStatusProvider` context，管理步骤状态 | ~80 行 |
| Phase 3 | 改造 `SkillApiDependencies` 和 `CustomParamsSection` 为内联 | ~100 行 |
| Phase 4 | 改造 `LifecycleControls` 加入完成度检查 | ~60 行 |
| Phase 5 | 后端 `enable_skill` 扩展验证（可选） | ~20 行 |
| **Total** | | **~410 行** |

**修复路径建议：** `story` — 这是一个 interaction/flow level 的 UI 重新设计，涉及多个文件和跨前后端改动。

---

*Bug analysis completed. See fix proposal above for implementation guidance.*
