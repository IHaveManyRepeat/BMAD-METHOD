# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260515-001 |
| **Title** | 系统上下文字段编辑策略缺失：未根据引用状态区分可编辑范围 |
| **Type** | missing-validation |
| **Severity** | High |
| **Status** | open |
| **Analyzed At** | 2026-05-15 |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 001 |

---

## 1. Bug Description

### 1.1 Summary

管理端 `system_context_fields` 的编辑功能存在两个层面的问题：

1. **后端** `UpdateFieldRequest` 仅包含 `description` 字段，无法修改 `field_name`、`field_type`、`options`
2. **后端** `update_field` 服务没有根据 `reference_count` 区分可编辑范围 — 对所有字段统一只允许修改 description
3. **前端** `EditFieldDialog` 将 `field_name` 和 `field_type` 硬编码为只读展示，`options` 完全不展示编辑控件

### 1.2 Expected Behavior

| 字段 | `reference_count == 0`（未引用） | `reference_count > 0`（已引用） |
|------|:---:|:---:|
| `field_name` | 可编辑 | 不可编辑 |
| `field_type` | 可编辑 | 不可编辑 |
| `description` | 可编辑 | 可编辑 |
| `options[].label` | 可编辑 | 可编辑 |
| `options[].value` | 可编辑 | 不可编辑 |

### 1.3 Actual Behavior

- 所有字段（无论引用状态）只能修改 `description`
- `field_name`、`field_type` 始终只读
- `options` 完全不可编辑

### 1.4 Root Cause

- `UpdateFieldRequest` (model.rs:101-104) 只定义了 `description` 字段
- `repo::update` (repo.rs:125-142) SQL 只更新 `description`
- `service::update_field` (service.rs:110-131) 没有查询 `reference_count` 做条件判断
- 前端 `EditFieldDialog` (EditFieldDialog.tsx) 硬编码所有字段为只读

---

## 2. Code Graph Analysis

### 2.1 Affected Files

```
admin-backend/src/features/system_context/
├── model.rs          # UpdateFieldRequest 定义 — 需扩展字段
├── service.rs        # update_field — 需增加 reference_count 判断逻辑
├── repo.rs           # update — 需支持多字段更新
└── routes.rs         # update_field_handler — 可能需要微调

admin-frontend/src/features/system-context/
├── types/index.ts              # UpdateFieldRequest 类型 — 需扩展
├── api/systemContextApi.ts     # API 调用 — 无需变更
├── components/EditFieldDialog.tsx  # 编辑对话框 — 需根据引用状态动态渲染
└── hooks/useSystemContext.ts   # hooks — 可能需要微调
```

### 2.2 Call Graph

```
PUT /api/admin/system-context-fields/{id}
  → routes::update_field_handler
    → service::update_field
      → repo::find_by_id              # 查询现有字段（含 reference_count）
      → [MISSING] reference_count check
      → repo::update                  # 目前只更新 description
        → SQL: UPDATE system_context_fields SET description=$1, updated_at=NOW()
```

### 2.3 Data Flow

```
前端 EditFieldDialog
  → systemContextApi.update(id, { description })    # 只传 description
    → PUT /api/admin/system-context-fields/{id}
      → UpdateFieldRequest { description }          # 只接受 description
        → service::update_field
          → repo::update(id, description)           # 只更新 description
```

---

## 3. Root Cause Analysis

### 3.1 Workflow-Level Cause

初始设计时为简化实现，`update` 接口仅支持修改 `description`。这在系统上下文字段刚创建、尚未被技能引用时是合理的（避免破坏引用完整性），但**过度限制**了未被引用字段的编辑自由度。

### 3.2 Implementation-Level Cause

| 层级 | 问题 | 位置 |
|------|------|------|
| Model | `UpdateFieldRequest` 缺少 `field_name`, `field_type`, `options` 字段 | model.rs:101-104 |
| Service | `update_field` 不查询 `reference_count`，无条件限制所有字段的修改 | service.rs:110-131 |
| Repo | `update` SQL 硬编码只更新 `description` 和 `updated_at` | repo.rs:125-142 |
| Frontend Type | `UpdateFieldRequest` TS 类型只含 `description` | types/index.ts:36-38 |
| Frontend UI | `EditFieldDialog` 硬编码字段名和类型为只读，options 完全不渲染编辑控件 | EditFieldDialog.tsx:63-78 |

### 3.3 Why This Matters

- **未被引用的字段**：管理员无法修改名称、类型、选项，必须删除后重建，体验差
- **已被引用的字段**：虽然当前的"只允许修改 description"策略安全，但缺少对 options label 的编辑支持，影响可维护性
- **select 类型字段**：options 的 label（展示文本）修改不影响数据完整性，但当前完全禁止编辑

---

## 4. Fix Proposal

### 4.1 Backend Changes

#### 4.1.1 Expand `UpdateFieldRequest` (model.rs)

```rust
/// PUT /api/admin/system-context-fields/{id}
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UpdateFieldRequest {
    pub field_name: Option<String>,
    pub field_type: Option<String>,
    pub description: Option<String>,
    /// For select type: full options array (label + value)
    /// When referenced, value changes are rejected by service layer
    pub options: Option<serde_json::Value>,
}
```

#### 4.1.2 Add reference-aware validation in service (service.rs)

```rust
pub async fn update_field(pool, id, req) -> Result<FieldResponse, AppError> {
    let existing = repo::find_by_id(pool, id).await?
        .ok_or_else(|| AppError::NotFound("..."))?;

    let is_referenced = existing.reference_count > 0;

    // Unreferenced: allow all changes
    // Referenced: reject field_name, field_type changes;
    //             allow description, options.label changes;
    //             reject options.value changes
    if is_referenced {
        if req.field_name.is_some() && req.field_name.as_deref() != Some(existing.field_name.as_str()) {
            return Err(AppError::Validation("被引用字段的名称不可修改"));
        }
        if req.field_type.is_some() && req.field_type.as_deref() != Some(existing.field_type.as_str()) {
            return Err(AppError::Validation("被引用字段的类型不可修改"));
        }
        if let Some(new_options) = &req.options {
            validate_options_value_unchanged(&existing, new_options)?;
        }
    } else {
        // Validate field_name if provided
        if let Some(name) = &req.field_name {
            validate_field_name(name)?;
            // Check uniqueness if name changed
            if name != existing.field_name {
                if repo::find_by_name(pool, name).await?.is_some() {
                    return Err(AppError::Conflict("字段名已存在"));
                }
            }
        }
        // Validate field_type if provided
        if let Some(ft) = &req.field_type {
            validate_field_type(ft)?;
        }
    }

    // Validate options for select type
    // ...

    let updated = repo::update_full(pool, id, resolved_fields).await?;
    Ok(FieldResponse::from_field(&updated))
}
```

#### 4.1.3 Add `update_full` to repo (repo.rs)

New function that updates all provided fields using COALESCE pattern:

```sql
UPDATE system_context_fields SET
    field_name = COALESCE($1, field_name),
    field_type = COALESCE($2, field_type),
    description = $3,
    options = COALESCE($4, options),
    updated_at = NOW()
WHERE id = $5
RETURNING *
```

### 4.2 Frontend Changes

#### 4.2.1 Expand `UpdateFieldRequest` type (types/index.ts)

```typescript
export interface UpdateFieldRequest {
  field_name?: string
  field_type?: FieldType
  description?: string
  options?: SelectOption[]
}
```

#### 4.2.2 Rework `EditFieldDialog` (EditFieldDialog.tsx)

- Accept full `SystemContextField` detail (with `reference_count`)
- If `reference_count === 0`: render all fields as editable inputs
- If `reference_count > 0`:
  - `field_name`: readonly (grayed out)
  - `field_type`: readonly (grayed out)
  - `description`: editable input
  - For select options: `label` editable, `value` readonly (grayed out)

### 4.3 Validation Plan

| Test Case | Input | Expected |
|-----------|-------|----------|
| Unreferenced field: change name | `field_name: "new_name"` | Success |
| Unreferenced field: change type | `field_type: "number"` | Success |
| Unreferenced field: change options | full new options array | Success |
| Referenced field: change description | `description: "new"` | Success |
| Referenced field: change name | `field_name: "new"` | 400 Validation Error |
| Referenced field: change type | `field_type: "number"` | 400 Validation Error |
| Referenced field: change option label | label changed, value unchanged | Success |
| Referenced field: change option value | value changed | 400 Validation Error |
| Referenced field: no changes to name/type | same name/type sent | Success (idempotent) |

---

## 5. Impact Assessment

| Dimension | Impact |
|-----------|--------|
| **Files changed** | ~6 files (3 backend + 3 frontend) |
| **Lines of code** | ~150-200 lines |
| **Risk level** | Medium — modifying core CRUD logic |
| **Backward compatibility** | Full — existing `description`-only calls still work |
| **Fix path** | story (medium change, multiple files, needs validation tests) |

---

*Report generated by bug analysis workflow.*
