# 工作流变更提案

---

## 当前问题

数据库表定义中缺少字段注释（COMMENT），导致：
1. 新成员无法快速理解表结构和字段用途
2. 代码审查时需要额外询问字段含义
3. 维护和重构时增加沟通成本
4. 缺少明确的编码规范

---

## 建议变更

### 1. 修改 Drizzle Schema 定义

**文件：** `packages/backend/src/shared/pg-schema.ts`

**变更内容：**

```typescript
// 为所有非技术字段添加 .comment()

// 示例变更：
const conversations = s.table("conversations", {
    id: uuid("id").primaryKey().defaultRandom(), // 技术字段，不注释
    title: text("title")
        .notNull()
        .comment("对话标题"), // ← 添加注释
    cellphone: text("cellphone")
        .comment("用户手机号"), // ← 添加注释
    accessToken: text("access_token")
        .comment("访问令牌"), // ← 添加注释
    platformSource: platformSourceEnum("platform_source")
        .notNull()
        .default("General")
        .comment("平台来源：Aigis/Mes/General"), // ← 添加注释
    // ... 其他字段类似处理
});
```

### 2. 重新生成 SQL 迁移脚本

**变更原因：** 旧迁移脚本没有 COMMENT 语句

**新脚本示例：**

```sql
-- Drizzle 会自动生成带 COMMENT 的 SQL
CREATE TABLE IF NOT EXISTS conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title TEXT NOT NULL, -- 对话标题
    cellphone TEXT, -- 用户手机号
    access_token TEXT, -- 访问令牌
    platform_source VARCHAR(20) NOT NULL DEFAULT 'General', -- 平台来源
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 3. 建立 Schema 定义规范

**创建新文件：** `packages/backend/src/shared/pg-schema-standards.md`

**内容：**
- 强制要求非技术字段使用 `.comment()`
- 列出所有表和必需注释的字段列表
- 定义注释编写指南（中文、简洁、说明业务用途）
- 禁止技术字段添加注释（id, created_at, updated_at 等）

### 4. 更新代码审查标准

**更新文件：** 项目文档或贡献指南

**新增内容：**
- Schema 定义审查必须检查字段注释完整性
- 不满足注释要求的 PR 不予合并
- 列出所有表和字段的注释规范

---

## 影响

| 方面 | 说明 |
|--------|------|
| **修改范围** | 8 个表定义文件 + 10 个迁移文件 |
| **代码修改** | 每个文件 < 100 行，符合 `plan` 路径 |
| **验证增加** | ESLint 规则 + 单元测试 + 集成测试 |
| **破坏性** | 无 — 仅添加注释，不改变表结构 |

---

## 迁移路径

### Phase 1: 建立规范

1. 创建 `pg-schema-standards.md` 定义编码规范
2. ESLint 规则配置和测试
3. 单元测试框架设置

### Phase 2: 修复用户端表

1. 修改 `packages/backend/src/shared/pg-schema.ts` 中 `createUserTables()` 函数
2. 运行 Drizzle 生成新的迁移脚本
3. 运行 ESLint 验证
4. 编写并运行单元测试

### Phase 3: 修复管理端表

1. 修改 `packages/backend/src/shared/pg-schema.ts` 中 `createAdminTables()` 函数
2. 重新生成所有管理端迁移脚本
3. 运行 ESLint 验证
4. 编写并运行单元测试

### Phase 4: 验证和部署

1. 集成测试验证数据库 COMMENT
2. 在测试环境执行迁移
3. 代码审查确认符合规范
