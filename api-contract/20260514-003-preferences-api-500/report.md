# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | 20260514-003 |
| **Title** | preferences API 返回 500 错误 (user_test schema 查询失败) |
| **Type** | api-contract |
| **Severity** | HIGH |
| **Status** | open |
| **Analyzed At** | 2026-05-14T09:30:00.000Z |
| **Project** | diy-a2ui-v1.2 |
| **Version** | 003 |

---

## 1. Bug Description

### 1.1 Summary

调用 `GET /api/preferences` 接口时返回 500 Internal Server Error，错误信息显示 SQL 查询失败：`select "field_name", "field_value" from "user_test"."user_preferences"`。问题根本原因是 `.env` 配置使用了 `user_test` schema，但该 schema 中 `user_preferences` 表可能不存在或结构不匹配。

### 1.2 Reproduction Steps

1. 启动后端服务（packages/backend）
2. 前端调用 `GET http://localhost:5173/api/preferences`
3. 接口返回 500 错误，耗时 4132ms
4. 检查服务端日志看到 SQL 错误：`Failed query: select "field_name", "field_value" from "user_test"."user_preferences"`

### 1.3 Expected Behavior

- `GET /api/preferences` 应正常返回用户的偏好设置
- SQL 查询应成功执行并返回数据

### 1.4 Actual Behavior

- 接口返回 HTTP 500 错误
- 错误响应：`{"ok":false,"error":{"code":"INTERNAL_ERROR","message":"Failed query: select \"field_name\", \"field_value\" from \"user_test\".\"user_preferences\"\nparams: ","traceId":"..."}}`
- 耗时 4132ms（较长，说明是数据库层错误）

---

## 2. Affected Areas

### 2.1 Affected Workflows

- `bmad-pipeline` — 部署/启动流程

### 2.2 Affected Files

| 文件 | 问题描述 |
|------|----------|
| `.env` | `DATABASE_SCHEMA=user_test` 使用了测试 schema |
| `packages/backend/src/routes/preferences.ts` | 调用 `getSchemaInstance().user.userPreferences` |
| `packages/backend/src/shared/pg-schema-instance.ts` | 初始化 schema 实例 |
| `admin-backend/migrations/0015_create_user_preferences.sql` | 迁移文件，定义 `user_preferences` 表结构 |

### 2.3 Configuration

```
# .env 中的当前配置
DATABASE_SCHEMA=user_test      # 用户端自身 schema
ADMIN_SCHEMA=admin_test        # 跨 schema 只读访问的管理端 schema
```

---

## 3. Root Cause Analysis

### 3.1 代码流程

```
GET /api/preferences
  → createPreferencesRoutes() [packages/backend/src/routes/preferences.ts:186]
  → deps.db.select({ fieldName, fieldValue }).from(userPreferences)
  → userPreferences 来自 getSchemaInstance().user
  → getSchemaInstance() 返回 pgSchema("user_test").table("user_preferences", ...)
  → 生成的 SQL: SELECT "field_name", "field_value" FROM "user_test"."user_preferences"
```

### 3.2 Schema 定义对比

**Drizzle ORM 定义** (`packages/backend/src/shared/pg-schema.ts:166-177`):
```typescript
const userPreferences = s.table("user_preferences", {
  id: uuid("id").primaryKey().defaultRandom(),
  fieldName: text("field_name").notNull(),
  fieldValue: text("field_value").notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, ...);
```

**迁移 SQL 定义** (`admin-backend/migrations/0015_create_user_preferences.sql`):
```sql
CREATE TABLE IF NOT EXISTS user_preferences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  field_name TEXT NOT NULL,
  field_value TEXT NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**列名对比**:
| ORM 映射 | Drizzle 列名 | SQL 列名 | 状态 |
|----------|-------------|----------|------|
| fieldName | "field_name" | field_name | ✓ 匹配 |
| fieldValue | "field_value" | field_value | ✓ 匹配 |
| updatedAt | "updated_at" | updated_at | ✓ 匹配 |

### 3.3 根本原因

**确认：配置与数据库迁移不同步**

`.env` 配置:
```
DATABASE_SCHEMA=user_test
```

问题是：
1. `user_test` schema 是开发/测试环境 schema
2. `0015_create_user_preferences.sql` 迁移文件需要在 `user_test` schema 中运行
3. 如果迁移未在 `user_test` 中运行，则表不存在导致 SQL 错误
4. 如果使用 `user_production` schema，表应该存在

### 3.4 代码图 (Code Graph)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         preferences.ts                               │
│                    GET /api/preferences (L186)                       │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼ deps.db.select()
┌─────────────────────────────────────────────────────────────────────┐
│                       pg-schema-instance.ts                          │
│              getSchemaInstance().user.userPreferences                 │
│           pgSchema("user_test").table("user_preferences")           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼ SQL: SELECT "field_name", "field_value"
┌─────────────────────────────────────────────────────────────────────┐
│                    PostgreSQL (user_test schema)                     │
│              "relation \"user_preferences\" does not exist"           │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.5 验证命令

在 PostgreSQL 中验证：
```sql
-- 检查 user_test schema 中的表
SELECT table_name FROM information_schema.tables 
WHERE table_schema = 'user_test';

-- 检查 user_preferences 表是否存在
SELECT * FROM user_test.user_preferences LIMIT 1;

-- 如果不存在，在 user_test 中创建
SET search_path TO user_test;
\i admin-backend/migrations/0015_create_user_preferences.sql
```

---

## 4. Fix Proposal

### 4.1 方案一：在 user_test schema 中创建表（推荐用于开发环境）

在 `user_test` schema 中运行迁移：

```sql
-- 连接到数据库后执行
SET search_path TO user_test;

-- 创建 user_preferences 表
CREATE TABLE IF NOT EXISTS user_preferences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  field_name TEXT NOT NULL,
  field_value TEXT NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX IF NOT EXISTS preference_field_unique_idx ON user_preferences(field_name);

-- 验证
SELECT * FROM user_preferences LIMIT 1;
```

**优点**：保留开发/测试配置不变
**缺点**：如果数据库重置，需要重新运行迁移

### 4.2 方案二：修改 .env 使用 production schema

```env
DATABASE_SCHEMA=user_production
ADMIN_SCHEMA=admin_production
```

**优点**：使用生产环境 schema，表已存在
**缺点**：在生产环境测试有风险

### 4.3 方案三：改进错误处理（防御性修复）

在 `preferences.ts` 中增加诊断信息，当表不存在时返回更有意义的错误：

```typescript
routes.get("/preferences", async (c) => {
  try {
    const { userPreferences } = getSchemaInstance().user;
    const rows = await deps.db
      .select({
        fieldName: userPreferences.fieldName,
        fieldValue: userPreferences.fieldValue,
      })
      .from(userPreferences);
    // ...
  } catch (err) {
    const message = err instanceof Error ? err.message : String(err);
    // 增强错误诊断
    if (message.includes("does not exist")) {
      logger.error({ 
        schema: config.databaseSchema,
        table: "user_preferences"
      }, "Table or column not found in database schema");
    }
    throw err; // 重新抛出，让 errorHandler 处理
  }
});
```

**优点**：未来遇到类似问题更容易诊断
**缺点**：不解决当前问题

---

## 5. Validation Plan

1. 修改 `.env` 后重启后端服务
2. 调用 `GET /api/preferences` 验证返回 200
3. 验证返回的用户偏好数据格式正确
4. 测试 `PUT /api/preferences` 保存偏好设置

---

## 6. Related Bugs

- `20260514-001` — 用户端凭证设置未读取管理端系统上下文配置的字段

---

*Bug analysis document created in Step 1. Code graph and root cause analysis will be appended in subsequent steps.*
