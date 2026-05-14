---

## 4. Prevention Strategy

### 4.1 Workflow Change

| Current | Proposed |
|---------|----------|
| Backend `fetchSchemaFromAdmin()` 失败时返回 `ok:true, data:[]`，静默降级 | Backend 应返回明确的错误状态（503 或 `unreachable:true`），让前端知道连接失败 |
| 前端 `PreferencesForm` 不消费 `warning` 字段 | 前端展示 `warning` 信息，让用户知道是连接问题而非真的无字段 |
| 字段定义为空时用户无法判断"真的没有"还是"连接失败" | 增加连接状态指示器（ConnectionStatus 组件） |

### 4.2 Automated Validation

1. **集成测试**：验证 admin API 不可达时 backend 返回 503 或 `unreachable:true`
2. **E2E 测试**：验证前端在 admin API 不可达时展示"无法连接管理端"提示
3. **单元测试**：`preferences.ts` 错误处理分支必须覆盖网络超时、401、500 三种场景

---

## 5. Fix Proposal

### 5.1 Fix Path

**Decision**: `story` — 涉及 backend 和 frontend 两侧的跨模块改动，需要修改 API 契约（新增 `unreachable` 字段或改 HTTP 状态码），影响范围 ≥ 2 文件

### 5.2 Bug Type

**Selected**: `api-contract` — 前后端 API 契约不完整，error handling 未同步

### 5.3 Concrete Fix Plan

**Story: 修复偏好设置管理端连接失败的静默降级问题**

**Backend 改动**（`packages/backend/src/routes/preferences.ts`）：

1. 修改 `GET /preferences/schema` 错误处理：
   - 网络错误 / 5xx → HTTP 503，`{ ok: false, error: { code: "ADMIN_UNREACHABLE", message: "无法连接管理端" } }`
   - 401/403 → HTTP 503，`{ ok: false, error: { code: "ADMIN_AUTH_FAILED", message: "管理端认证失败" } }`
   - 仅在 cache 有效时返回缓存数据（即使 stale）

2. 新增 `unreachable` 响应字段（可选方案B）：
   ```typescript
   return c.json({
     ok: true,
     data: cachedSchema ?? [],
     warning: "无法连接管理端，暂无字段定义",
     unreachable: true,  // 新增字段
   });
   ```

**Frontend 改动**（`packages/frontend/src/chat/components/PreferencesForm.tsx`）：

1. `useEffect` 中检测 `schemaResult` 失败：
   ```typescript
   if (schemaResult.status === "rejected") {
     setSyncWarning("无法连接管理端，请检查网络或联系管理员");
     // 或直接显示连接失败状态
   }
   ```

2. `getPreferenceSchema()` 返回类型扩展，支持解析 `warning` 字段

**验证要求**：
- [ ] `preferences.ts` 单元测试：网络超时场景 → HTTP 503
- [ ] `preferences.ts` 单元测试：401 场景 → HTTP 503
- [ ] `PreferencesForm.tsx` E2E：管理端不可达时显示错误提示而非空表单
- [ ] 回归测试：正常情况下偏好表单仍正常显示

---

## 6. Automated Verification Mechanism

### 6.1 Unit Tests

- [ ] `preferences.test.ts` — `fetchSchemaFromAdmin` 超时 → 503
- [ ] `preferences.test.ts` — `fetchSchemaFromAdmin` 401 → 503
- [ ] `preferences.test.ts` — `fetchSchemaFromAdmin` 成功但空数组 → 200 + 空数组

### 6.2 Integration Tests

- [ ] 启动 backend + mock admin API（返回 200 但空数组）→ `/api/preferences/schema` 返回 200
- [ ] mock admin API down → `/api/preferences/schema` 返回 503

### 6.3 E2E Tests

- [ ] 打开设置面板 → admin API 正常 → 显示字段表单
- [ ] 打开设置面板 → admin API 不可达 → 显示"无法连接管理端"提示

### 6.4 Static Analysis

- [ ] `PreferencesForm.tsx` 警告未使用 `syncWarning` 状态（ESLint）
- [ ] `preferences.ts` 确认无 `unwrap()` / `expect()`

### 6.5 Runtime Checks

- [ ] Backend 启动时验证 `ADMIN_SERVICE_TOKEN` 非空否则 warn 日志

---

*Prevention Strategy, Fix Proposal, and Verification Mechanism added in Step 4.*
**✅ Bug Analysis Report is now complete and can be used for coding!*