# Bug Analysis Report: 禁用用户时 422 错误

## 元数据

| 字段 | 值 |
|--------|-----|
| Bug ID | 20260427-v1.2-user-status-422 |
| 发现日期 | 2026-04-27 |
| 严重级别 | High |
| 影响模块 | admin-backend, admin-frontend |
| Bug 类型 | API Contract (接口契约问题) |

---

## Bug 描述

### 问题摘要

在管理端尝试禁用用户时，API 调用返回 `422 Unprocessable Entity` 错误，导致用户状态无法更新。

### 复现步骤

1. 打开管理端用户管理页面
2. 找到目标用户并点击"禁用"按钮
3. 前端发送 `PUT /api/admin/users/{id}` 请求，携带 `{ "is_active": false }`
4. 后端返回 `HTTP/1.1 422 Unprocessable Entity`
5. 用户状态保持不变

### 预期行为

用户状态应成功更新为禁用（`is_active: false`）。

### 实际行为

API 返回 `422 Unprocessable Entity`，用户状态未更新。

---

## 受影响文件

| 文件路径 | 角色 |
|----------|------|
| `admin-backend/src/features/user/model.rs` | 后端类型定义 |
| `admin-backend/src/features/user/routes.rs` | API 路由处理器 |
| `admin-backend/src/features/user/service.rs` | 业务逻辑 |
| `admin-frontend/src/features/users/api/userApi.ts` | 前端 API 客户端 |
| `admin-frontend/src/features/users/types/index.ts` | 前端类型定义 |
| `admin-frontend/e2e/users.spec.ts` | E2E 测试 |

---

## 代码图分析

### 调用链

```
前端用户操作
    ↓
admin-frontend/src/features/users/api/userApi.ts::userApi.update()
    ↓
PUT /api/admin/users/{id} { "is_active": false }
    ↓
admin-backend/src/features/user/routes.rs::update_user_handler
    ↓
service::update_user()
    ↓
UpdateUserRequest { username: String } ← 反序列化失败！
```

### 类型不匹配

| 组件 | 定义 | 字段 |
|--------|--------|------|
| 前端 | `UpdateUserRequest` | `is_active?: boolean, password?: string` |
| 后端 | `UpdateUserRequest` | `username: String` (必填) |

### 孤立组件

- `PATCH /api/admin/users/{id}/status` 端点存在但未被前端使用
- `UpdateUserStatusRequest` 类型存在但未被前端使用

---

## 根本原因分析

### 1. Bug 分类

**API Contract（接口契约问题）** — 前后端接口定义不一致

### 2. 直接原因

**反序列化失败**：前端发送的请求体（`{ "is_active": false }`）无法反序列化为后端的 `UpdateUserRequest { username: String }` 结构，导致 Axum 返回 422 错误。

### 3. 深层根本原因

#### 设计阶段问题

1. **缺乏 Design-First 流程**：
   - 后端定义了两个端点：
     - `PUT /api/admin/users/{id}` → 更新用户名
     - `PATCH /api/admin/users/{id}/status` → 启用/禁用用户
   - 前端实现时未遵循后端设计，错误地使用 PUT 端点

2. **类型定义不同步**：
   - 前端 `UpdateUserRequest` 包含 `is_active` 和 `password` 字段
   - 后端 `UpdateUserRequest` 仅包含 `username` 字段
   - 没有机制确保类型定义一致

3. **测试覆盖不足**：
   - E2E 测试未包含用户禁用/启用场景
   - 缺少 API 契约验证测试

### 4. 工作流级别原因

| 问题 | 描述 |
|------|------|
| 缺失验证步骤 | 没有 TypeScript-to-Rust 类型一致性检查 |
| 缺少测试要求 | E2E 测试未覆盖用户状态管理 |
| 缺失文档 | 没有 API 使用指南区分不同端点 |

### 5. 防御策略

| 策略 | 具体措施 |
|--------|----------|
| Design-First | 使用 OpenAPI 3.x 规范，前后端从同一规范生成类型 |
| 自动化验证 | CI 中添加接口契约一致性检查 |
| E2E 测试 | 添加用户禁用/启用的端到端测试 |
| 文档同步 | 前后端类型变更时更新 API 文档 |

---

## 修复方案

### 修复路径

**Plan** — 直接修复单个接口定义，不需要创建完整 story。

理由：
- 影响文件数：2 个（`userApi.ts`, `types/index.ts`）
- 代码变更量：~20 行
- 测试补充：1 个 E2E 测试用例

### 具体修复

#### 1. 前端 API 客户端修正

**文件：** `admin-frontend/src/features/users/api/userApi.ts`

```diff
 export const userApi = {
   list: async (): Promise<User[]> => {
     const response = await apiClient.get<UserListResponse>('/users')
     return response.data.users
   },

   getById: async (id: string): Promise<User> => {
     const response = await apiClient.get<User>(`/users/${id}`)
     return response.data
   },

   create: async (data: CreateUserRequest): Promise<User> => {
     const response = await apiClient.post<User>('/users', data)
     return response.data
   },

   update: async (id: string, data: UpdateUserRequest): Promise<User> => {
     const response = await apiClient.put<User>(`/users/${id}`, data)
     return response.data
   },

+  updateStatus: async (id: string, isActive: boolean): Promise<User> => {
+    const response = await apiClient.patch<User>(`/users/${id}/status`, { is_active: isActive })
+    return response.data
+  },

   delete: async (id: string): Promise<void> => {
     await apiClient.delete(`/users/${id}`)
   },
 }
```

#### 2. 前端类型定义修正

**文件：** `admin-frontend/src/features/users/types/index.ts`

```diff
 export interface User {
   id: string
   username: string
   is_active: boolean
   failed_login_attempts: number
   locked_until: string | null
   created_at: string
   updated_at: string
 }

 export interface CreateUserRequest {
   username: string
   password: string
 }

-export interface UpdateUserRequest {
-  is_active?: boolean
-  password?: string
+  username: string  // 与后端一致：仅更新用户名
 }
+
+export interface UpdateUserStatusRequest {
+  is_active: boolean
+}
```

#### 3. E2E 测试补充

**文件：** `admin-frontend/e2e/users.spec.ts`

```typescript
test('disables user successfully', async ({ page }) => {
  // 1. 登录
  await login(page)

  // 2. 导航到用户页面
  await page.goto('/users')
  await page.waitForSelector('[data-testid="users-list"]')

  // 3. 点击第一个用户的禁用按钮
  await page.click('[data-testid="disable-user-button"]:first-child')

  // 4. 验证确认对话框
  await expect(page.locator('[data-testid="confirm-disable-dialog"]')).toBeVisible()

  // 5. 确认禁用
  await page.click('[data-testid="confirm-disable-button"]')

  // 6. 验证用户状态已更新为禁用
  await expect(page.locator('[data-testid="user-status-badge"]:first-child')).toHaveText('已禁用')
  await expect(page.locator('[data-testid="disable-user-button"]:first-child')).toBeHidden()
  await expect(page.locator('[data-testid="enable-user-button"]:first-child')).toBeVisible()
})

test('enables user successfully', async ({ page }) => {
  // 1. 登录
  await login(page)

  // 2. 导航到用户页面，找到已禁用的用户
  await page.goto('/users')
  await page.waitForSelector('[data-testid="users-list"]')

  // 3. 点击启用按钮
  await page.click('[data-testid="enable-user-button"]:first-child')

  // 4. 验证用户状态已更新为启用
  await expect(page.locator('[data-testid="user-status-badge"]:first-child')).toHaveText('已启用')
  await expect(page.locator('[data-testid="enable-user-button"]:first-child')).toBeHidden()
  await expect(page.locator('[data-testid="disable-user-button"]:first-child')).toBeVisible()
})
```

---

## 验证计划

### 单元测试

- [ ] 后端 `UpdateUserStatusRequest` 反序列化验证
- [ ] 前端 `updateStatus` API 方法测试

### 集成测试

- [ ] `PATCH /api/admin/users/{id}/status` 端点测试
- [ ] 禁用超级管理员应返回 403
- [ ] 禁用不存在的用户应返回 404

### E2E 测试

- [ ] 用户禁用流程
- [ ] 用户启用流程
- [ ] 错误提示显示正确

### 静态分析

- [ ] TypeScript 编译无错误
- [ ] Rust clippy 无警告

### 运行时检查

- [ ] API 返回正确的状态码
- [ ] 错误消息用户友好

---

## 工作流变更提议

### 当前问题

前端和后端 API 契约不一致，导致用户禁用功能失败。

### 提议变更

采用 **Design-First** 流程：

1. 使用 OpenAPI 3.x 规范定义 API
2. 从 OpenAPI 生成前端 TypeScript 类型
3. 从 OpenAPI 生成后端 Rust 验证
4. CI 中添加接口契约一致性检查

### 影响

- 需要更新 API 调用代码
- 需要添加测试覆盖
- 不会破坏现有功能

### 迁移路径

1. **紧急**：修复当前 bug（本报告）
2. **短期**：补充 E2E 测试覆盖
3. **中期**：考虑采用 OpenAPI-First 架构

---

## 相关文档

- `admin-backend/src/features/user/model.rs` — 后端类型定义
- `admin-frontend/src/features/users/api/userApi.ts` — 前端 API 调用
- `CLAUDE.md` — 项目技术栈和架构决策
