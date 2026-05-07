# Automated Validation Plan

---

## Bug Reference

- **Bug ID**: 20260506-004
- **Bug Type**: ui-ux (major)
- **Bug Title**: 接口详情页的参数定义缺乏编辑功能

---

## 1. Unit Tests

### 1.1 Test Cases

| # | Test File | Test Name | Validates | Priority |
|---|-----------|-----------|-----------|----------|
| 1 | ApiForm.test.tsx | 表单支持 params_schema 编辑 | P0 |
| 2 | ApiForm.test.tsx | 编辑时 params_schema 正确传递到 API | P0 |
| 3 | ApiForm.test.tsx | 表单支持 response_schema 编辑 | P1 |
| 4 | ApiForm.test.tsx | 编辑时 response_schema 正确传递到 API | P1 |

### 1.2 Test Code Template

```typescript
describe('ApiForm 组件 - 参数编辑功能', () => {
  it('应该支持 params_schema 编辑', () => {
    // Arrange
    const mockApi: ApiRegistration = {
      id: 'test-id',
      api_alias: 'test-api',
      description: 'test desc',
      http_method: 'POST',
      path: '/api/test',
      environment_id: 'env-1',
      params_schema: { params: [{ name: 'param1', in: 'body', required: true }] },
      auth_type: 'none',
      status: 'active',
      version: 1,
      created_at: '2024-01-01',
      updated_at: '2024-01-01',
    };

    // Act
    const { container } = render(<ApiForm open editItem={mockApi} onOpenChange={vi.fn()} />);
    // 查找 params_schema 输入控件

    // Assert
    // 验证 params_schema 编辑器存在
    const paramsEditor = container.querySelector('[data-testid="params-schema-editor"]');
    expect(paramsEditor).toBeInTheDocument();
  });

  it('编辑时应该正确传递 params_schema 到 payload', async () => {
    // Arrange
    const mockApi: ApiRegistration = {
      id: 'test-id',
      api_alias: 'test-api',
      description: 'test desc',
      http_method: 'POST',
      path: '/api/test',
      environment_id: 'env-1',
      params_schema: { params: [{ name: 'param1', in: 'body', required: true }] },
      auth_type: 'none',
      status: 'active',
      version: 1,
      created_at: '2024-01-01',
      updated_at: '2024-01-01',
    };
    const mockUpdate = vi.fn().mockResolvedValue(mockApi);
    (useUpdateApiRegistration as unknown as ReturnType<typeof vi.mock>).mockReturnValue({
      mutateAsync: mockUpdate,
      isPending: false,
    });

    const { container } = render(<ApiForm open editItem={mockApi} onOpenChange={vi.fn()} />);
    const paramsEditor = container.querySelector('[data-testid="params-schema-editor"]');
    const form = container.querySelector('form');

    // Act
    // 修改 params_schema
    fireEvent.change(paramsEditor!, { target: { value: '{"params": [{"name": "newParam", "in": "body"}]}' } });
    await fireEvent.submit(form!);

    // Assert
    expect(mockUpdate).toHaveBeenCalledWith({
      id: 'test-id',
      data: expect.objectContaining({
        params_schema: { params: [{ name: 'newParam', in: 'body' }] },
      }),
    });
  });
});
```

---

## 2. Integration Tests

| # | Scenario | Files | Validates |
|---|----------|-------|-----------|
| 1 | 完整编辑流程 | ApiForm.spec.ts, useApiRegistrations.test.ts | 编辑接口的 params_schema 后数据正确更新到数据库 |

---

## 3. E2E Tests

| # | Flow | Steps | Validates |
|---|------|-------|-----------|
| 1 | 接口参数编辑流程 | 3 steps | 用户可以通过编辑表单修改参数定义 |

### E2E 测试步骤：
1. 访问接口详情页
2. 点击"编辑"按钮
3. 修改参数定义 JSON
4. 保存
5. 验证参数定义已更新

---

## 4. Static Analysis

### 4.1 Lint Rules

```yaml
# .eslintrc.js 或对应配置
rules:
  react/prop-types: error
  @typescript-eslint/explicit-module-boundary-types: error
```

### 4.2 Type Checks

- [ ] ApiForm 组件 props 类型完整包含 params_schema 字段
- [ ] params_schema state 变量类型定义正确
- [ ] handleSubmit payload 类型包含 params_schema

---

## 5. Runtime Checks

### 5.1 Input Validation

```typescript
// validateParamsSchema 函数
function validateParamsSchema(input: string): { valid: boolean; error?: string } {
  if (!input.trim()) {
    return { valid: true }; // 空值允许
  }

  try {
    const parsed = JSON.parse(input);
    if (typeof parsed !== 'object' || parsed === null) {
      return { valid: false, error: '参数定义必须是有效的 JSON 对象' };
    }

    // 验证 params 结构（如果需要）
    if (parsed.params && !Array.isArray(parsed.params)) {
      return { valid: false, error: 'params 必须是数组' };
    }

    return { valid: true };
  } catch (e) {
    return { valid: false, error: '无效的 JSON 格式' };
  }
}
```

### 5.2 Output Validation

```typescript
// ApiForm 组件保存后验证
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  setError('');

  // ... 现有验证

  // 新增：params_schema 验证
  if (paramsSchemaInput.trim()) {
    const validation = validateParamsSchema(paramsSchemaInput);
    if (!validation.valid) {
      setError(validation.error || '参数定义格式错误');
      return;
    }
  }

  try {
    const payload = {
      // ... 现有字段
      params_schema: paramsSchemaInput.trim() ? JSON.parse(paramsSchemaInput) : undefined,
    };

    await updateApi.mutateAsync({ id: editItem.id, data: payload });
    onOpenChange(false);
  } catch (err) {
    const message = err instanceof Error ? err.message : '操作失败';
    setError(message);
  }
};
```

---

## 6. CI Integration

### 6.1 Pipeline Gate

```yaml
# 添加到 CI 配置确保测试通过
stages:
  - name: Test
    jobs:
      - name: Unit Tests
        run: pnpm test
      - name: Type Check
        run: pnpm exec tsc --noEmit
```

### 6.2 Verification Script

```bash
# 验证 ApiForm 组件的字段完整性
echo "Verifying ApiForm field coverage..."

# 检查 ApiForm.tsx 是否包含 params_schema 相关代码
if ! grep -q "params_schema" admin-frontend/src/features/api-registrations/components/ApiForm.tsx; then
  echo "✓ params_schema editing is implemented"
else
  echo "✗ params_schema editing is missing"
  exit 1
fi
```

---

*Generated by `bmad-analyze-bug` workflow*
