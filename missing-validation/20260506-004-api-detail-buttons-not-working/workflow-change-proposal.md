# Workflow Change Proposal — 防止空壳按钮进入生产

## 当前问题

管理端存在 6 个"空壳按钮"（有 UI 但无实际功能），全部通过了 CI 测试。核心原因是测试策略只覆盖了数据层（hook），没有覆盖 UI 层（组件按钮行为），且 E2E 测试大量使用软断言跳过不存在的元素。

## 提议变更

### 变更 1：组件级测试强制要求

**当前行为：** 仅 hook 层有单元测试，组件层无测试
**提议行为：** 每个包含交互按钮的页面组件必须有对应的测试文件，覆盖所有按钮点击行为

**测试模板：**
```typescript
// 每个页面组件必须验证的操作
describe('PageComponent buttons', () => {
  it('每个 Button 必须有实际行为', () => {
    render(<PageComponent />)
    const buttons = screen.getAllByRole('button')
    // 排除 type="submit" 和 disabled 的按钮
    const actionButtons = buttons.filter(b => !b.hasAttribute('disabled') && b.type !== 'submit')
    // 每个按钮点击后必须有副作用（API 调用 / 状态变更 / 导航）
    for (const btn of actionButtons) {
      // 验证按钮不是空壳
    }
  })
})
```

### 变更 2：E2E 测试硬断言标准

**当前行为：** `if (await button.isVisible()) { ... }` — 按钮不存在时测试静默跳过
**提议行为：** 核心操作使用硬断言，缺失即为失败

**修改规则：**
```typescript
// ❌ 禁止：软断言（核心操作）
const btn = page.getByRole('button', { name: '编辑' })
if (await btn.isVisible()) { await btn.click() }

// ✅ 要求：硬断言
await expect(page.getByRole('button', { name: '编辑' })).toBeVisible()
await page.getByRole('button', { name: '编辑' }).click()
```

**例外：** 仅"未来功能"按钮（如导入 Swagger）可以使用软断言，但必须在代码中以注释标记。

### 变更 3：TODO 追踪机制

**当前行为：** `// TODO: Implement toggle status` 无追踪
**提议行为：**

1. 所有 TODO 必须关联 GitHub Issue
2. 合并前必须决定：**实现功能** 或 **移除按钮 UI**（不发布空壳按钮）
3. CI 检查：`grep -r "TODO" src/ | grep -i "button\|click\|handle"` 列出待处理项

### 变更 4：代码审查 Checklist 增加

```markdown
## 按钮功能审查（新增）

- [ ] 所有 `<Button>` / `<button>` 元素有明确的功能行为
- [ ] 无 `onClick={() => {}}` 空函数（设计稿占位除外）
- [ ] 无 `console.log` 作为按钮处理函数
- [ ] 每个新增按钮有对应的测试覆盖
```

## 影响

| 方面 | 变更前 | 变更后 |
|------|--------|--------|
| 开发速度 | 快（跳过测试） | 略慢（必须写组件测试） |
| 测试可靠性 | 假绿灯（软断言跳过） | 真绿灯（硬断言验证） |
| Bug 检出 | 依赖人工发现 | 自动化检测 |
| 空壳按钮风险 | 高 | 极低 |

## 迁移路径

1. 先修复当前 6 个空壳按钮
2. 为 API 注册、用户、环境三个核心模块补充组件测试
3. 逐步将 E2E 软断言改为硬断言
4. 建立 ESLint 规则自动检测
5. 更新 CR checklist
