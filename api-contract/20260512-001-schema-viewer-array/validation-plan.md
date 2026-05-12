# Validation Plan — SchemaViewer Bug Fix

## Unit Tests

### Test File: `admin-frontend/src/features/api-registrations/__tests__/SchemaViewer.test.tsx`

| Test Case | Description |
|-----------|-------------|
| `flattenSchemaProperties_handlesParamsArray` | 测试 `params_schema` 格式 `{ params: [...] }` 能正确解析 |
| `flattenSchemaProperties_handlesJsonSchemaObject` | 测试标准 JSON Schema 格式 `{ type: 'object', properties: {...} }` |
| `flattenSchemaProperties_handlesArrayWithImplicitObjectItems` | 测试数组 items 只有 `properties` 没有 `type: 'object'` 的情况 |
| `flattenSchemaProperties_handlesNestedArrays` | 测试嵌套数组情况 |
| `flattenSchemaProperties_handlesEmptySchema` | 测试空 schema 返回空数组 |
| `flattenSchemaProperties_handlesMixedTypes` | 测试混合类型数组 |

### Test Data Examples

```typescript
// Bug 1: params_schema 格式
const paramsSchema = {
  params: [
    { name: 'id', type: 'string', in: 'path', required: true },
    { name: 'data', type: 'object', in: 'body', required: true }
  ]
};

// Bug 2: array items 没有 type 字段
const responseSchema = {
  type: 'object',
  properties: {
    data: {
      type: 'array',
      items: {
        properties: {
          id: { type: 'string' },
          name: { type: 'string' }
        }
      }
    }
  }
};
```

## Integration Tests

| Test Case | Description |
|-----------|-------------|
| `SchemaViewer_inApiDetailPage` | 在 ApiDetailPage 中验证 schema 显示正确 |
| `SchemaViewer_withRealApiData` | 使用真实 API 数据验证显示 |

## Static Analysis

- 添加 TypeScript 类型检查确保 `SchemaField` 接口正确
- ESLint 规则检查无未处理的类型

## Runtime Checks

- Console warning 当 schema 格式无法解析时
- Fallback 到 JSON 源码视图当结构视图为空时
