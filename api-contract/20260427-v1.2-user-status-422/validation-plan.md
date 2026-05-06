# Validation Plan: 禁用用户 422 错误修复

## 目标

确保用户禁用/启用功能正确工作，并防止未来接口契约不一致问题。

---

## 验证清单

### 1. 代码变更验证

#### 前端 API 客户端
- [ ] `userApi.updateStatus()` 方法已添加
- [ ] 方法调用 `PATCH /users/${id}/status` 端点
- [ ] 请求体格式正确：`{ is_active: boolean }`
- [ ] 返回类型为 `Promise<User>`

#### 前端类型定义
- [ ] `UpdateUserRequest` 仅包含 `username: string`
- [ ] `UpdateUserStatusRequest` 已添加，包含 `is_active: boolean`
- [ ] 所有字段与后端定义一致

#### 组件集成
- [ ] 用户管理页面使用 `updateStatus()` 方法
- [ ] 禁用按钮触发 `updateStatus(id, false)`
- [ ] 启用按钮触发 `updateStatus(id, true)`
- [ ] 错误处理显示用户友好消息

### 2. 测试验证

#### 单元测试
- [ ] 前端 `updateStatus` API 方法测试通过
- [ ] 后端 `UpdateUserStatusRequest` 反序列化测试通过

#### 集成测试
- [ ] `PATCH /api/admin/users/{id}/status` 返回 200 成功
- [ ] 禁用超级管理员返回 403 Forbidden
- [ ] 禁用不存在的用户返回 404 Not Found
- [ ] 请求体验证正确处理无效 JWT

#### E2E 测试
- [ ] 用户禁用流程完整通过
- [ ] 用户启用流程完整通过
- [ ] 错误提示显示正确（用户友好）
- [ ] UI 状态更新反映禁用/启用结果

### 3. 静态分析

#### TypeScript
- [ ] `tsc --noEmit` 无错误
- [ ] 无类型断言错误
- [ ] 无隐式 any 类型

#### Rust
- [ ] `cargo clippy` 无警告
- [ ] `cargo fmt` 格式正确
- [ ] `cargo test` 所有测试通过

### 4. 运行时检查

#### API 响应验证
- [ ] 成功响应：`200 OK` + `User` 对象
- [ ] 错误响应格式正确
- [ ] 响应时间为 < 500ms

#### 错误处理
- [ ] 网络错误显示友好提示
- [ ] 422 错误不再出现
- [ ] 错误消息不暴露技术细节

---

## 自动化验证机制

### CI/CD 检查

```yaml
# .github/workflows/verify-api-contract.yml
name: Verify API Contract

on: [pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Check TypeScript Types
        run: |
          cd admin-frontend
          pnpm tsc --noEmit

      - name: Check Rust Types
        run: |
          cd admin-backend
          cargo clippy -- -D warnings

      - name: Run E2E Tests
        run: |
          cd admin-frontend
          pnpm test:e2e
```

### 本地验证脚本

```bash
#!/bin/bash
# verify-fix.sh

echo "=== 验证 Bug 修复 ==="

# 1. TypeScript 类型检查
echo "1. 检查前端类型..."
cd admin-frontend
if ! pnpm tsc --noEmit; then
  echo "❌ TypeScript 类型检查失败"
  exit 1
fi

# 2. Rust 类型检查
echo "2. 检查后端类型..."
cd ../admin-backend
if ! cargo clippy -- -D warnings; then
  echo "❌ Rust clippy 检查失败"
  exit 1
fi

# 3. E2E 测试
echo "3. 运行 E2E 测试..."
cd ../admin-frontend
if ! pnpm test:e2e; then
  echo "❌ E2E 测试失败"
  exit 1
fi

echo "✅ 所有验证通过！"
```

---

## 回归测试检查

### 功能验证

- [ ] 创建用户功能正常
- [ ] 更新用户名功能正常
- [ ] 重置密码功能正常
- [ ] 删除用户功能正常
- [ ] 用户列表显示正常

### 性能验证

- [ ] 用户列表加载 < 1s
- [ ] 状态更新响应 < 500ms
- [ ] 无 N+1 查询问题

### 安全验证

- [ ] 未授权访问被拒绝
- [ ] 超级管理员保护生效
- [ ] JWT 验证正确
- [ ] 输入验证生效

---

## 验收标准

### 必须通过（Must Have）

1. ✅ 用户禁用功能正常工作
2. ✅ 用户启用功能正常工作
3. ✅ 无 422 错误出现
4. ✅ E2E 测试全部通过

### 应该通过（Should Have）

1. ⚠️ 代码覆盖率 > 80%
2. ⚠️ 错误提示用户友好
3. ⚠️ 无 TypeScript/Rust 警告

### 可以通过（Nice to Have）

1. 💡 添加性能监控
2. 💡 添加 API 契约自动化验证
3. 💡 生成 OpenAPI 规范

---

## 回归预防

### 类型定义同步机制

1. **创建类型映射文档**：
   ```
   docs/api-contract-mapping.md
   - 前端接口 → 后端结构体映射
   - 端点用途说明
   - 更新日期和责任人
   ```

2. **添加变更检查**：
   - 前端类型变更 → 验证后端定义
   - 后端结构体变更 → 验证前端类型
   - 端点新增/删除 → 更新文档

3. **定期审查**：
   - 每周进行类型一致性检查
   - 每月进行完整 API 契约审计

### 测试覆盖要求

- [ ] 所有 API 端点有集成测试
- [ ] 所有用户流程有 E2E 测试
- [ ] 测试覆盖率 > 80%
- [ ] 关键路径有回归测试
