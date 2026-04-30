# Validation Plan

## Validation Plan: 数据库字段注释缺失

---

## 1. 静态分析

| 工具 | 规则 | 描述 |
|------|------|------|
| ESLint | 禁止缺失注释的表定义 | Drizzle `.table()` 中，除 `id` 等纯技术字段外，所有其他字段必须使用 `.comment()` |
| TypeScript 编译 | 类型检查 | 确保 `.comment()` 方法存在且类型正确 |

### 验证方法

1. 运行 ESLint 检查 `pg-schema.ts` 文件
2. 使用 `drizzle-kit` 生成 SQL 时验证包含 COMMENT

---

## 2. 单元测试

### 测试文件

| 测试文件 | 测试内容 |
|----------|----------|
| `pg-schema.test.ts` | 验证每个表定义都包含 `.comment()` 调用 |

### 测试场景

1. **conversations 表**：验证 title, cellphone, accessToken 等字段有注释
2. **messages 表**：验证 role, content, toolCalls 等字段有注释
3. **user_settings 表**：验证 userId, settingKey, settingValue 等字段有注释
4. **environments 表**：验证 name, baseUrl, commonHeaders 等字段有注释
5. **api_registrations 表**：验证 apiAlias, httpMethod, path 等字段有注释
6. **skill_dependencies 表**：验证 skillId, apiId, status 等字段有注释
7. **audit_logs 表**：验证 operatorId, operationType, targetId 等字段有注释
8. **test_histories 表**：验证 apiId, userId, parameters 等字段有注释
9. **users 表**：验证 username, passwordHash 等字段有注释

---

## 3. 集成测试

### 测试场景

| 测试 | 验证内容 |
|------|----------|
| 数据库查询 | 使用 `information_schema.columns` 查询验证 COMMENT 存在 |

### 验证 SQL

```sql
SELECT
    table_name,
    column_name,
    data_type,
    is_nullable,
    column_default,
    comment
FROM information_schema.columns
WHERE table_schema IN ('admin_production', 'user_production')
ORDER BY table_name, ordinal_position;
```

---

## 4. 验证清单

- [ ] ESLint 规则已创建并执行
- [ ] 单元测试覆盖所有表定义
- [ ] 生成 SQL 包含 COMMENT 语句
- [ ] 集成测试验证数据库 COMMENT 存在
- [ ] 所有非技术字段都有业务语义注释
- [ ] 技术字段（id, created_at 等）保持无注释（可选）
