# Bug Analysis Report: 骨架文件重新生成缺少二次确认

| Field | Value |
|-------|-------|
| **ID** | 20260525-006 |
| **Date** | 2026-05-25 |
| **Type** | ui-ux |
| **Severity** | Medium |
| **Status** | Fixed |
| **Fix Path** | plan (minor UI-UX, < 20 lines, 1 file) |

---

## Bug Description

技能详情页的"骨架生成"步骤中，点击"重新生成"按钮会立即执行骨架文件覆盖操作，无任何确认提示。重新生成会覆盖已有的骨架代码文件（包含用户可能已做的自定义修改），属于不可逆的破坏性操作。

### Reproduction Steps

1. 导航到任意技能详情页
2. 切换到"骨架生成"步骤
3. 先点击"生成骨架"生成一份骨架代码
4. 点击"重新生成"按钮
5. **期望**：弹出确认对话框，提示用户覆盖风险
6. **实际**：直接覆盖，无任何确认

### Affected Areas

- `admin-frontend/src/features/skills/components/SkillConfigStepper.tsx` — `SkeletonSection` 组件，第 159-226 行

---

## Code Graph Analysis

### Call Chain

```
用户点击"重新生成"按钮
  → handleGenerate() (直接调用，无拦截)
    → generateMutation.mutate(skillId)
      → skillsApi.generateSkeleton(skillId)
        → POST /api/skills/:id/skeleton
          → 后端 skeleton_generator 覆盖文件
```

### Key Symbols

| Symbol | Location | Role |
|--------|----------|------|
| `SkeletonSection` | `SkillConfigStepper.tsx:159` | 骨架生成步骤的 UI 组件 |
| `handleGenerate` | `SkillConfigStepper.tsx:167` | 生成/重新生成的 mutation 调用 |
| `useGenerateSkeleton` | `useSkills.ts:330` | React Query mutation hook |
| `skillsApi.generateSkeleton` | `skillsApi.ts` | API 请求函数 |

### Problem

`handleGenerate` 函数被"生成骨架"（首次）和"重新生成"两个按钮共享调用。首次生成无需确认，但重新生成（已有 `skeleton` 数据时）属于覆盖操作，应增加确认环节。两个按钮的调用路径完全一致，缺少分支判断。

---

## Root Cause Analysis

### Workflow Level

无相关工作流问题。这是一个典型的 UI 防护缺失。

### Implementation Level

**直接原因**：`SkeletonSection` 组件中"重新生成"按钮直接绑定 `handleGenerate`，未加任何确认逻辑。

```tsx
// 修改前 — 按钮直接调用 handleGenerate
<Button onClick={handleGenerate} disabled={generateMutation.isPending}>重新生成</Button>
```

**根本原因**：开发时未区分"首次生成"与"重新生成"两个操作的风险差异。首次生成是安全操作（创建新文件），重新生成是破坏性操作（覆盖已有文件），两者应有不同的交互流程。

**项目已有的同类模式**：项目已引入 `AlertDialog` 组件（`admin-frontend/src/components/ui/alert-dialog.tsx`），基于 Radix UI 的 AlertDialog primitive，专门用于此类确认场景。但 `SkeletonSection` 未使用。

---

## Fix

### Changes Made

**文件**: `admin-frontend/src/features/skills/components/SkillConfigStepper.tsx`

| Change | Lines | Description |
|--------|-------|-------------|
| 新增 AlertDialog 导入 | +9 | 导入 AlertDialog 系列组件 |
| 新增 `showRegenerateConfirm` 状态 | +1 | 控制确认对话框显隐 |
| 新增 `handleRegenerateClick` 方法 | +6 | 有骨架时弹确认，无骨架时直接生成 |
| "重新生成"按钮改绑 | 1 | `onClick` 从 `handleGenerate` → `handleRegenerateClick` |
| 新增 AlertDialog 确认框 | +15 | 含标题、风险描述、取消/确认按钮 |

### Behavior After Fix

| 场景 | 行为 |
|------|------|
| 首次点击"生成骨架" | 直接执行，无确认 |
| 点击"重新生成" | 弹出 AlertDialog 确认框 |
| 确认框中点击"取消" | 关闭对话框，不执行任何操作 |
| 确认框中点击"确认重新生成" | 执行覆盖生成 |

### Code Diff Summary

```diff
+ import { AlertDialog, AlertDialogContent, ... } from '@/components/ui/alert-dialog'

  function SkeletonSection({ skillId, onGenerated }: SkeletonSectionProps) {
    const [generatedSkeleton, setGeneratedSkeleton] = useState<SkillSkeleton | null>(null)
+   const [showRegenerateConfirm, setShowRegenerateConfirm] = useState(false)
    const generateMutation = useGenerateSkeleton()

+   const handleRegenerateClick = () => {
+     if (skeleton) {
+       setShowRegenerateConfirm(true)
+     } else {
+       handleGenerate()
+     }
+   }

    // 骨架代码展示区域
    <Button onClick={handleRegenerateClick}>重新生成</Button>
+   <AlertDialog open={showRegenerateConfirm} onOpenChange={setShowRegenerateConfirm}>
+     <AlertDialogContent>
+       <AlertDialogTitle>确认重新生成骨架文件？</AlertDialogTitle>
+       <AlertDialogDescription>
+         重新生成将覆盖当前的骨架代码文件，已有的自定义修改将丢失。此操作不可撤销。
+       </AlertDialogDescription>
+       <AlertDialogCancel>取消</AlertDialogCancel>
+       <AlertDialogAction onClick={handleGenerate}>确认重新生成</AlertDialogAction>
+     </AlertDialogContent>
+   </AlertDialog>
  }
```

---

## Validation

### Manual Test Checklist

- [ ] 首次生成骨架：点击"生成骨架"按钮，直接执行，无弹框
- [ ] 重新生成：点击"重新生成"按钮，弹出 AlertDialog 确认框
- [ ] 确认框内容：标题和描述文字准确说明覆盖风险
- [ ] 取消操作：点击"取消"关闭对话框，骨架代码不变
- [ ] 确认操作：点击"确认重新生成"执行覆盖，loading 状态正确
- [ ] 骨架不存在时：按钮显示"生成骨架"，点击直接生成
- [ ] 无 TypeScript 编译错误
- [ ] 无 ESLint 警告

### Automated Validation

无需自动化测试 — 此为纯 UI 交互行为，适合手动验证。
