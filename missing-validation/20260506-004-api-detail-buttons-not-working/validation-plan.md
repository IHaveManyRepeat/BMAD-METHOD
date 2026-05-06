# Validation Plan — API 详情页按钮不生效

## 目标

确保管理端所有按钮具备可验证的功能行为，建立从机制上防止"空壳按钮"进入生产代码的测试体系。

---

## 1. 直接修复验证

### 1.1 ApiDetailPage 编辑按钮

| # | 测试类型 | 测试描述 | 预期结果 | 优先级 |
|---|----------|----------|----------|--------|
| V1 | 单元测试 | 点击编辑按钮后 formOpen 变为 true | `setFormOpen(true)` 被调用 | P0 |
| V2 | 单元测试 | 点击编辑按钮后 editItem 被设置为当前 api | `setEditItem(api)` 被调用 | P0 |
| V3 | 单元测试 | ApiForm 打开后 editItem 有值时显示"编辑接口"标题 | 对话框标题为"编辑接口" | P1 |
| V4 | E2E 测试 | 详情页点击编辑 → 对话框弹出 → 修改字段 → 保存 → 数据更新 | 修改后的值出现在详情页 | P0 |

### 1.2 ApiDetailPage 禁用/启用按钮

| # | 测试类型 | 测试描述 | 预期结果 | 优先级 |
|---|----------|----------|----------|--------|
| V5 | 单元测试 | 点击禁用按钮后调用 useUpdateApiRegistration | `mutateAsync({id, data: {status: 'inactive'}})` 被调用 | P0 |
| V6 | 单元测试 | 点击启用按钮后调用 useUpdateApiRegistration | `mutateAsync({id, data: {status: 'active'}})` 被调用 | P0 |
| V7 | 单元测试 | mutation 成功后 query cache 被刷新 | `invalidateQueries` 被调用 | P1 |
| V8 | E2E 测试 | 详情页点击禁用 → 状态变为 inactive → 刷新后保持 | StatusBadge 显示 Inactive | P0 |

---

## 2. 系统性验证

### 2.1 管理端按钮功能完整性矩阵

每个 CRUD 模块的标准测试模板：

```typescript
// 标准模板：每个模块必须覆盖的 5 个核心操作
describe('CRUD Operations Smoke Test', () => {
  test('创建：点击新建按钮 → 表单弹出 → 填写并提交 → 列表出现新记录')
  test('读取：列表页展示数据 → 点击行进入详情 → 详情页显示完整信息')
  test('更新：详情页点击编辑 → 表单弹出预填数据 → 修改并保存 → 数据更新')
  test('禁用/启用：点击禁用 → 状态变更 → 点击启用 → 状态恢复')
  test('删除：点击删除 → 确认对话框 → 确认后记录消失')
})
```

### 2.2 空壳按钮检测

| 检测方式 | 检测内容 | 触发时机 |
|----------|----------|----------|
| ESLint 规则 | `<Button` / `<button` 无 `onClick`、非 `type="submit"`、非 `disabled` | lint 阶段 |
| ESLint 规则 | `onClick={() => {}}` 空函数 | lint 阶段 |
| ESLint 规则 | `onClick` 处理函数体仅为 `console.log` | lint 阶段 |
| 组件测试 | 每个按钮的 `onClick` mock 被调用 | 测试阶段 |
| E2E 测试 | 核心操作按钮使用硬断言 `await expect(button).toBeEnabled()` | CI 阶段 |

### 2.3 测试覆盖率目标

| 模块 | 当前覆盖 | 目标覆盖 |
|------|----------|----------|
| API 注册 | 0 个组件测试 + 5 个 E2E（软断言） | 3 个组件测试 + 8 个 E2E（硬断言） |
| 用户管理 | 0 个组件测试 + 4 个 E2E（软断言） | 2 个组件测试 + 5 个 E2E（硬断言） |
| 环境管理 | 0 个组件测试 + 4 个 E2E（软断言） | 2 个组件测试 + 5 个 E2E（硬断言） |
| 全局 Hook 层 | 已覆盖 | 保持 |

---

## 3. 验证执行顺序

1. **先修复 bug**：实现 ApiDetailPage 编辑/禁用功能
2. **补充组件测试**：编写 ApiDetailPage.test.tsx
3. **补充 E2E 测试**：添加详情页操作 E2E
4. **建立 ESLint 规则**：检测空壳按钮
5. **审计其余空壳按钮**：逐个修复或标记为"未来功能"
