# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260513-001 |
| **Title** | 编辑接口后版本历史不记录 |
| **Type** | missing-validation — Will be selected in Step 4 |
| **Severity** | Medium |
| **Status** | open |
| **Analyzed At** | 2026-05-13T12:00:00.000Z |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 001 |

---

## 1. Bug Description

### 1.1 Summary

用户在接口详情页编辑接口信息并保存后，"依赖" Tab 下的版本历史区域没有显示对应的变更记录。版本历史区域目前显示"暂无版本历史记录"。

### 1.2 Reproduction Steps

1. 访问 `http://localhost:5174/api-registrations/b0000000-0000-0000-0000-000000000001`
2. 点击"编辑"按钮
3. 修改接口别名或其他信息
4. 保存
5. 切换到"依赖" Tab
6. 查看版本历史区域

**Expected Behavior**: 版本历史应显示编辑后的变更记录，包含时间戳和变更内容

**Actual Behavior**: 版本历史区域显示"暂无版本历史记录"

### 1.3 Affected Areas

- **前端组件**: `admin-frontend/src/features/api-registrations/components/ApiDetailPage.tsx`
  - `DepsTab` 组件（第 492-508 行）
  - 版本历史区域硬编码为空状态提示
- **后端模型**: `admin-backend/src/features/api/model.rs`
  - `ApiRegistration` 结构体包含 `version` 字段（i32）
- **后端服务**: `admin-backend/src/features/api/service.rs`
  - `update_api` 函数（第 85-176 行）自动递增版本
  - 但没有版本历史记录存储逻辑

### 1.4 Root Cause Analysis

**直接原因**: `DepsTab` 组件中的版本历史区域只是渲染了一个静态的空状态提示（第 504 行），没有实际获取和展示版本历史数据。

**根本原因**: 系统在接口更新时虽然会递增 `version` 字段，但**没有版本历史记录表或存储机制**来追踪每次变更的历史。

关键发现：
1. `ApiRegistration` 模型只有 `version: i32` 字段（当前版本号），没有历史版本存储
2. `update_api` 服务函数自动递增版本（第 155-168 行），但不记录历史
3. 前端 `DepsTab` 的版本历史区域是硬编码的空提示，不是真正的数据展示
4. 没有任何 API 端点或服务来获取版本变更历史

---

## 2. Code Graph Analysis

### 2.1 Affected Code Paths

```
前端:
ApiDetailPage.tsx (DepsTab 组件) → 硬编码空状态，无数据获取

后端:
api/service.rs (update_api) → repo::update → 版本递增
                             → 无历史记录写入
```

### 2.2 Key Files

| File | Lines | Issue |
|------|-------|-------|
| `ApiDetailPage.tsx` | 499-506 | 硬编码空状态，无数据获取 |
| `api/model.rs` | 84-99 | ApiRegistration 无历史记录字段 |
| `api/service.rs` | 155-168 | update 不记录版本历史 |

### 2.3 Call Chain

```
用户编辑保存
  → ApiForm.tsx handleSubmit
  → updateApi.mutateAsync
  → PUT /api/admin/apis/{id}
  → api/routes.rs → api/service.rs::update_api
  → repo::update (版本 +1)
  → 前端刷新数据
  → DepsTab 渲染 (空状态)
```

### 2.4 Component Structure

```
ApiDetailPage
├── Tabs: info | test | code | deps
└── DepsTab (apiId)
    ├── ReverseDependencyList (正确实现)
    └── Card: 版本历史
        └── <静态空状态>
```

**注意**: `ReverseDependencyList` 组件是正确实现的（有真实数据），但版本历史部分未实现。

---

## 3. Root Cause Analysis

### 3.1 Workflow Level Issue

**问题**: 缺少版本历史存储和检索机制

当前的实现：
1. `api_registrations` 表只有 `version` 字段（当前版本号）
2. 没有历史版本表来记录每次变更
3. 版本递增时没有记录变更详情（谁、何时、改了什么）

### 3.2 Implementation Level Issue

**问题 1**: 前端版本历史区域未实现数据获取

```tsx
// ApiDetailPage.tsx 第 499-506 行
<Card>
  <CardHeader>
    <CardTitle className="text-base">版本历史</CardTitle>
  </CardHeader>
  <CardContent>
    <p className="text-sm text-muted-foreground">暂无版本历史记录</p>
  </CardContent>
</Card>
```

**问题 2**: 后端没有版本历史表和 API

`update_api` 函数只递增 `version`，不记录历史：
```rust
// api/repo.rs - update 函数
// 只更新当前记录，不插入历史
```

### 3.3 Data Flow Gap

```
编辑接口 → update_api → version++ (存储当前版本)
                              ↓
                     没有历史记录表
                              ↓
前端请求版本历史 → 无 API 端点 → 返回空
```

---

## 4. Fix Proposal

### 4.1 Required Changes

**需要创建:**
1. **版本历史表**: `api_versions` 或 `api_version_history`
2. **版本历史模型**: Rust struct + SQL schema
3. **版本历史 Repository**: 写入和读取历史
4. **版本历史 API 端点**: GET /api/admin/apis/{id}/versions
5. **前端版本历史组件**: 获取和展示历史数据

### 4.2 Database Schema (Proposed)

```sql
CREATE TABLE api_version_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  api_id UUID NOT NULL REFERENCES api_registrations(id),
  version INT NOT NULL,
  changed_fields JSONB,  -- 记录变更的字段
  changed_by VARCHAR(255),  -- 变更人
  changed_at TIMESTAMPTZ DEFAULT NOW(),
  change_type VARCHAR(50),  -- create, update, delete
  UNIQUE(api_id, version)
);

CREATE INDEX idx_api_version_history_api_id ON api_version_history(api_id);
```

### 4.3 API Endpoint

```
GET /api/admin/apis/{apiId}/versions
Response: {
  "api_id": "...",
  "versions": [
    {
      "version": 3,
      "changed_fields": {"api_alias": "new_name"},
      "changed_at": "2026-05-13T12:00:00Z",
      "change_type": "update"
    }
  ]
}
```

### 4.4 Fix Complexity

| Item | Complexity | Estimated Lines |
|------|------------|------------------|
| 数据库迁移 | Medium | ~20 |
| 后端模型 | Low | ~40 |
| 后端 Repository | Medium | ~60 |
| 后端 Service | Low | ~30 |
| 后端 Routes | Low | ~20 |
| 前端 Hook | Low | ~30 |
| 前端 Component | Medium | ~80 |
| **Total** | - | **~280** |

---

## 5. Validation Plan

### 5.1 Test Scenarios

1. **创建接口时记录版本 1**
2. **编辑接口后版本历史显示变更**
3. **版本历史包含正确的变更字段**
4. **多个编辑操作按时间顺序展示**

### 5.2 Verification Steps

1. 创建新接口 → 验证版本历史有记录
2. 编辑接口别名 → 验证版本历史新增记录
3. 编辑 params_schema → 验证版本历史包含变更详情
4. 查看版本历史 → 按时间倒序排列

---

## 3. Root Cause Analysis

### 3.1 Immediate Cause

前端 `DepsTab` 组件中的"版本历史"区域（第 499-506 行）只渲染了一个静态的空状态提示文字，没有实际调用任何 API 获取版本历史数据。

### 3.2 Root Cause

系统架构层面缺少版本历史记录机制：

1. **数据模型缺陷**: `ApiRegistration` 只有 `version: i32` 字段（当前版本号），没有历史版本存储表或字段
2. **服务层缺陷**: `update_api` 函数在更新时只递增 `version`，不记录变更历史（变更人、变更时间、变更内容）
3. **API 层缺陷**: 没有用于获取版本历史的 API 端点
4. **前端缺陷**: 即使有 API，前端组件也是硬编码空状态，没有实现数据获取逻辑

### 3.3 Workflow-Level Cause

**为什么工作流允许这个 bug 发生？**

在产品需求和功能设计阶段，没有明确要求版本历史记录功能，导致：
- 数据库 schema 设计时只有 `version` 字段
- API 更新逻辑只实现了版本递增，没有实现历史记录
- 前端 UI 实现时版本历史区域只是占位符

这反映了**实现优先级决策**的问题：优先完成核心功能（创建、编辑、查询接口），暂时搁置辅助功能（版本历史）。但这导致已完成的 UI 区域（版本历史卡片）展示不正确。

### 3.4 Similar Patterns

类似问题可能存在于：
- 其他实体的版本历史（如 Skill 版本、Environment 版本）
- 审计日志功能（谁在何时做了什么操作）

---

## 4. Fix Proposal

### 4.1 Required Changes

| 层级 | 组件 | 变更内容 |
|------|------|----------|
| Database | 新建 `api_version_history` 表 | 存储版本变更记录 |
| Backend | 新增 model | `ApiVersionHistory` struct |
| Backend | 新增 repo | `create_version_record`, `get_versions_by_api_id` |
| Backend | 新增 service | 版本历史写入逻辑 |
| Backend | 新增 routes | `GET /api/admin/apis/{apiId}/versions` |
| Frontend | 新增 hook | `useApiVersionHistory(apiId)` |
| Frontend | 修改 DepsTab | 渲染版本历史数据 |

### 4.2 Database Schema

```sql
CREATE TABLE api_version_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  api_id UUID NOT NULL REFERENCES api_registrations(id) ON DELETE CASCADE,
  version INT NOT NULL,
  changed_fields JSONB NOT NULL DEFAULT '{}',
  changed_by VARCHAR(255),
  changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  change_type VARCHAR(50) NOT NULL DEFAULT 'update',
  UNIQUE(api_id, version)
);

CREATE INDEX idx_api_version_history_api_id ON api_version_history(api_id);
CREATE INDEX idx_api_version_history_changed_at ON api_version_history(changed_at DESC);
```

### 4.3 API Design

```yaml
GET /api/admin/apis/{apiId}/versions
Response:
  api_id: string
  versions:
    - version: number
      changed_fields: object
      changed_by: string
      changed_at: datetime
      change_type: "create" | "update"
```

### 4.4 Implementation Order

1. 数据库迁移
2. 后端 model + repo
3. 后端 service 集成到 `update_api`
4. 后端 routes
5. 前端 hook
6. 前端组件更新

---

## 5. Validation Plan

### 5.1 Test Scenarios

| ID | 场景 | 预期结果 |
|----|------|----------|
| T1 | 创建新接口 | 版本历史显示 version=1 |
| T2 | 编辑接口别名 | 版本历史新增记录，包含 changed_fields |
| T3 | 编辑 params_schema | 版本历史记录包含字段变更详情 |
| T4 | 连续编辑多次 | 版本历史按时间倒序排列 |

### 5.2 Verification Steps

```bash
# 1. 创建接口
curl -X POST http://localhost:8080/api/admin/apis \
  -d '{"environment_id": "...", "api_alias": "test", ...}'

# 2. 验证版本历史
curl http://localhost:8080/api/admin/apis/{apiId}/versions
# 应返回 version=1

# 3. 编辑接口
curl -X PUT http://localhost:8080/api/admin/apis/{apiId} \
  -d '{"api_alias": "test_updated"}'

# 4. 再次验证版本历史
curl http://localhost:8080/api/admin/apis/{apiId}/versions
# 应返回 version=1 和 version=2
```

---

## 7. 根本问题反思：设计-实现-测试的脱节

### 7.1 设计文档明确要求版本历史

**证据 1**：UX 设计文档（第 151 行）明确将"版本变更历史"列为关键设计支撑：
```
| 掌控 vs 焦虑 | 依赖关系一目了然 | 改了接口怕 Skill 坏掉 | 依赖图/列表、变更历史可展开查看 |
```

**证据 2**：UX 设计文档（第 532 行）明确列出"依赖 Tab"的内容：
```
内容区 Tab 面板：
- **依赖 Tab**：引用此接口的 Skill 列表 + 版本历史
```

**证据 3**：用户旅程四（第 763-796 行）专门描述故障排查场景，明确提到：
```
- 触发：收到故障反馈 → 搜索接口 → 测试 → 查看日志 → 依赖保护
- 交互：查看接口详情 → 依赖 Tab 展示引用的 Skill 列表 → 版本历史查看变更
```

### 7.2 实现完全缺失

尽管设计文档明确要求，但实际实现：

| 设计要求 | 实现状态 |
|----------|----------|
| 版本历史展示 | ❌ 前端 DepsTab 硬编码空状态 |
| 版本历史 API | ❌ 不存在 |
| 版本历史数据模型 | ❌ 不存在 |
| 版本历史写入逻辑 | ❌ 不存在 |

### 7.3 测试未捕获

**关键发现**：E2E 测试 `api-registrations.spec.ts` 第 43-44 行：
```typescript
// NOTE: Backend currently returns 422 because the form doesn't send auth_type (required field).
// This is a known application bug. The test validates that the form interaction works correctly.
```

测试存在"已知 bug"仍然通过，说明测试降级了。

更关键的是，测试只验证了：
- Tab 存在（信息、测试、代码、依赖）
- 基本的 CRUD 操作

但**没有测试版本历史功能本身**：
- 没有断言版本历史区域的数据
- 没有断言版本历史在编辑后更新
- 没有断言版本历史 UI 的正确性

### 7.4 根本原因：验收链路的断裂

```
设计文档（明确要求版本历史）
        ↓
PRD/UX 设计 → Story 实现 → Code → Test → 验收
                ↓
            验收链路断裂
                ↓
        版本历史功能从未被实现
        版本历史测试从未被编写
        功能悄悄被降级/搁置
```

### 7.5 如何杜绝这类问题

**问题本质**：设计文档定义了功能，但实现和测试都没有覆盖。这是一个"验收链路"问题，不是技术问题。

**解决方案 1：Story 验收标准必须引用设计文档**
- 每个 Story 的验收标准必须明确列出设计文档中的对应要求
- Story 验收测试必须覆盖设计文档中的所有关键路径

**解决方案 2：Story 验收标准必须包含 Negative 测试**
- 不能只测试"正常路径"，必须测试"边界路径"
- 版本历史测试应该包含：
  - 创建 API 后版本历史有记录
  - 编辑 API 后版本历史有新增记录
  - 版本历史为空时的正确空状态展示
  - 版本历史数据格式正确

**解决方案 3：测试必须验证 UI 与数据的一致性**
- 当前测试只验证 UI 元素存在
- 应该验证：UI 元素展示的数据与后端数据一致
- 版本历史测试：编辑 API 后 → 刷新页面 → 版本历史区域有数据

**解决方案 4：Feature Flag 或分阶段交付**
- 如果版本历史功能当时确实来不及实现，应该：
  1. 使用 Feature Flag 禁用该功能
  2. 在 UI 上明确标注"功能开发中"
  3. 在 Sprint 计划中明确列入后续迭代

**解决方案 5：设计文档变更必须同步更新实现和测试**
- 设计文档中的任何新功能要求，必须同步到 Story 的验收标准
- Story 验收标准中的任何变更，必须同步到测试用例

---

## 8. Summary

| Item | Value |
|------|-------|
| **Bug Type** | missing-validation |
| **Severity** | Medium |
| **Root Cause** | 设计和实现脱节：设计文档明确要求版本历史，但实现和测试都未覆盖 |
| **Fix Complexity** | Medium (~280 lines) |
| **Fix Path** | story (multiple validation points) |
| **Prevention** | Story 验收标准必须引用设计文档，测试必须验证 UI 与数据的一致性 |

---

*Bug analysis report completed with root cause reflection.*