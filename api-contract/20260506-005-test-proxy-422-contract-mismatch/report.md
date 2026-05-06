# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260506-005 |
| **Title** | Test Proxy 请求 422 — 前后端契约不匹配 + 测试未覆盖 |
| **Type** | api-contract |
| **Severity** | High |
| **Status** | fixed |
| **Analyzed At** | 2026-05-06 |
| **Project** | diy-a2ui v1.2 |

---

## 1. Bug 描述

### 1.1 Summary

通过管理端 `ApiDetailPage` 内联测试或 `TestProxyPage` 测试 `GetMassifList` API 时，无论使用正确还是错误的参数名，均返回 422 错误。根因是前端 `apiRegistrationApi.test()` 发送的请求体结构与后端 Rust `TestProxyRequest` 结构完全不匹配。

### 1.2 Reproduction Steps

1. 登录管理端
2. 进入任意 API 详情页 → 切换到"测试"Tab
3. 输入参数 `{"cellPhone": "18612335173"}`
4. 点击"发送"
5. 返回 `500 Error` + `{"error": "Request failed with status code 422"}`

### 1.3 Expected Behavior

后端正确反序列化请求，代理转发到外部 API，返回 `TestProxyResponse { status_code, response_body, latency_ms }`。

### 1.4 Actual Behavior

后端反序列化失败（缺少必填字段 `environment_id`），返回 422。前端 Axios 将 422 包装为 `{"error": "Request failed with status code 422"}`。

---

## 2. 根因分析

### 2.1 直接原因：请求体结构不匹配

| 层级 | 期望 (后端 `TestProxyRequest`) | 实际 (前端 `apiRegistrationApi.test`) |
|------|------|------|
| 字段名 | `environment_id: Uuid` (必填) | **缺失** |
| 字段名 | `parameters: serde_json::Value` | `params` (错误字段名) |
| 字段名 | `auth_token: String` | 缺失 |
| 字段名 | `remember_auth: bool` | 缺失 |

**Bug 代码** (`apiRegistrationApi.ts:36-38`)：

```typescript
// ❌ 错误：发送 { params: {...} }，缺少 environment_id
test: async (id: string, params?: Record<string, unknown>): Promise<unknown> => {
    const response = await apiClient.post<unknown>(`/apis/${id}/test`, { params })
    return response.data
},
```

**后端期望** (`model.rs:31-42`)：

```rust
pub struct TestProxyRequest {
    pub environment_id: Uuid,       // ← 必填，反序列化失败
    #[serde(default)]
    pub parameters: serde_json::Value,  // ← 注意字段名是 parameters 不是 params
    #[serde(default)]
    pub auth_token: String,
    #[serde(default)]
    pub remember_auth: bool,
}
```

### 2.2 贡献因素：`return type: Promise<unknown>` 绕过类型安全

```typescript
// ❌ Promise<unknown> 让 TypeScript 放弃了对响应结构的类型检查
test: async (id: string, params?: Record<string, unknown>): Promise<unknown> => {
```

如果返回类型是 `Promise<TestProxyResponse>`，则：
- Mock 数据 `{ status: 200, data: { result: 'ok' } }` 会在编译时报错（字段名不匹配）
- 开发者能更早发现请求/响应结构不对

### 2.3 二级 Bug：参数名大小写（已在前一轮修复）

即使用了正确的请求格式，用户在 textarea 输入 `cellphone`（全小写）而非 `cellPhone`（PascalCase），后端 `distribute_params` 做 HashMap 精确匹配会失败。此问题通过添加参数提示 UI 解决。

---

## 3. 测试覆盖率缺口分析

### 3.1 现有测试清单

| 测试文件 | 覆盖范围 | 是否覆盖 Test Proxy |
|----------|----------|-------------------|
| `useApiRegistrations.test.tsx` | Hook 层 CRUD + test 调用 | ✅ 覆盖了 `useTestApiRegistration` 调用 |
| `ApiDetailPage.test.tsx` | 组件层编辑/启用/禁用 | ❌ **未覆盖测试 Tab** |
| `e2e/api-registrations.spec.ts` | E2E CRUD + 导航 | ❌ **未覆盖测试流程** |
| 后端 `service.rs` tests | 参数分发、URL 构建、Auth 注入 | ✅ 覆盖了后端纯函数 |

### 3.2 为什么现有测试没发现 Bug

**缺口 1：Hook 测试验证了错误行为**

```typescript
// useApiRegistrations.test.tsx (修复前)
it('TC-011: 成功测试接口调用', async () => {
    const testResponse = { status: 200, data: { result: 'ok' } }  // ← 假响应结构
    vi.mocked(apiRegistrationApi.test).mockResolvedValueOnce(testResponse)

    result.current.mutate({ id: 'api-uuid-1', params: { key: 'value' } })
    // ← 缺少 environmentId

    expect(apiRegistrationApi.test).toHaveBeenCalledWith('api-uuid-1', { key: 'value' })
    // ← 验证了错误的 2 参数签名，而非正确的 3 参数签名
})
```

问题链：
1. Mock 验证了 `apiRegistrationApi.test(id, params)` 被调用 — **这是错误的签名**
2. Mock 返回了 `{ status: 200, data: {...} }` — **这不是 `TestProxyResponse` 格式**
3. 因为返回类型是 `Promise<unknown>`，TypeScript 不会报错
4. 测试通过 ✅ 但验证的是错误行为

**缺口 2：组件测试完全未覆盖测试功能**

`ApiDetailPage.test.tsx` 只有 4 个测试（T1-T4），全部是编辑/启用/禁用操作。**测试 Tab 的点击、参数输入、发送请求、响应展示完全无覆盖**。

**缺口 3：E2E 测试未覆盖端到端测试流程**

`api-registrations.spec.ts` 有 6 个测试，覆盖 CRUD 和导航。无任何测试进入详情页 → 切换到测试 Tab → 发送请求 → 验证响应。

**缺口 4：Mock 隔离层过高，未验证 HTTP 契约**

所有测试都在 `apiRegistrationApi` 模块级别 Mock，从未验证：
- 实际发送的 HTTP body 结构
- URL path 是否正确
- 请求方法是否正确
- 响应是否按 `TestProxyResponse` 格式解析

### 3.3 缺口根因总结

```
                        测试金字塔

    E2E          ┌─────────────────┐
                 │  ❌ 未覆盖       │  缺少：测试 Tab → 填参 → 发送 → 验证响应
                 └─────────────────┘

  Integration    ┌─────────────────┐
                 │  ❌ 未覆盖       │  缺少：验证 HTTP 请求体匹配 TestProxyRequest
                 └─────────────────┘

    Unit         ┌─────────────────┐
                 │  ⚠️ 覆盖但验证   │  Hook 测试存在，但验证了错误的契约
                 │    了错误行为     │  + 返回类型 unknown 绕过类型安全
                 └─────────────────┘

  Contract       ┌─────────────────┐
                 │  ❌ 不存在       │  缺少：前后端接口契约测试
                 └─────────────────┘
```

---

## 4. 修复内容

### 4.1 代码修复

| 文件 | 修改 |
|------|------|
| `apiRegistrationApi.ts` | `test()` 添加 `environmentId` 参数，字段名 `params` → `parameters`，返回类型 `Promise<unknown>` → `Promise<TestProxyResponse>` |
| `useApiRegistrations.ts` | `useTestApiRegistration` 增加 `environmentId` 传递 |
| `ApiDetailPage.tsx` | 传递 `selectedEnvId`，正确解析 `TestProxyResponse` 的 `status_code`/`latency_ms`/`response_body` |
| `TestProxyPage.tsx` | 添加参数提示 + 填充模板（方案 A） |
| `ApiDetailPage.tsx` | 同步添加参数提示 + 填充模板 |

### 4.2 测试修复

| 文件 | 修改 |
|------|------|
| `useApiRegistrations.test.tsx` | Mock 响应改为 `TestProxyResponse` 格式，调用签名增加 `environmentId` |

### 4.3 仍需补充的测试

| 优先级 | 测试 | 类型 |
|--------|------|------|
| **P0** | ApiDetailPage 测试 Tab：填参 → 发送 → 验证调用 `apiRegistrationApi.test(id, envId, params)` | Unit |
| **P0** | ApiDetailPage 测试 Tab：验证 `selectedEnvId` 为空时按钮 disabled | Unit |
| **P1** | E2E：详情页 → 测试 Tab → 发送请求 → 验证响应展示 | E2E |
| **P1** | `apiRegistrationApi.test` 契约测试：验证 HTTP body 包含 `environment_id` + `parameters` | Integration |

---

## 5. 教训与预防措施

### 5.1 编码规范

1. **API 函数返回类型禁止 `unknown`** — 必须使用具体的 Response 类型，让 TypeScript 成为契约守门人
2. **前后端共享类型定义** — `TestProxyRequest`/`TestProxyResponse` 应从共享类型包导入，或通过 OpenAPI 生成
3. **新增 API 调用必须对照后端 struct** — 写 `apiClient.post` 时必须同时查看后端 `Deserialize` struct

### 5.2 测试规范

1. **Mock 不应验证实现细节** — 应验证"发送了正确格式的请求"而非"调用了某个函数"
2. **组件测试必须覆盖所有 Tab/路由** — ApiDetailPage 的 4 个 Tab 都应有测试
3. **E2E 必须覆盖核心用户流程** — "测试接口"是管理端核心功能，必须有 E2E 覆盖
4. **契约测试优先** — 前后端接口应有 schema 级别的契约验证，而非仅靠 Mock
