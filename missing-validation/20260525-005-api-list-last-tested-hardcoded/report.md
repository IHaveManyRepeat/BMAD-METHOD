# Bug: 接口注册页表格"最后测试"列硬编码显示横杠

| 字段 | 值 |
|------|------|
| ID | 20260525-005 |
| 日期 | 2026-05-25 |
| 严重程度 | **Medium** — 功能缺失，用户无法快速判断接口是否测试过及测试结果 |
| 影响范围 | 接口注册列表页所有 API 行 |
| 状态 | fixed |
| 分类 | missing-validation |

## 1. 错误现象

接口注册页（`/api-registrations`）表格的"最后测试"列对所有 API 均显示 `—`（横杠），即使该 API 已经通过测试代理执行过测试。用户无法从列表页了解：
- 接口是否被测试过
- 最近一次测试的状态码
- 最近一次测试的响应延迟

## 2. 根因分析

### 2.1 数据流断层

```
test_histories 表（已有完整测试记录）
    │
    ▼
api/repo::find_all()                 ← 只查询 api_registrations 表，未关联 test_histories
    │ 返回 ApiRegistration（无测试信息）
    ▼
api/service::list_apis()             ← 直接映射为 ApiResponse，无测试字段
    │
    ▼
ApiResponse                          ← 无 last_tested_at 等字段
    │
    ▼
前端 apiRegistrationApi.list()       ← 响应中无测试数据
    │
    ▼
ApiList.tsx:209-211                  ← 硬编码 —
```

### 2.2 两端缺失

1. **前端硬编码**（`admin-frontend/src/features/api-registrations/components/ApiList.tsx:209-211`）：
   ```tsx
   <TableCell className="text-xs text-muted-foreground">
     —
   </TableCell>
   ```
   直接写死横杠，未使用任何数据字段。

2. **后端 API 不返回测试信息**（`admin-backend/src/features/api/model.rs`）：
   `ApiResponse` 结构体中没有 `last_tested_at` 或相关字段。

3. **测试历史数据已存在但未被利用**：`test_histories` 表（migration 0004）记录了完整的测试历史（`created_at`、`response_status`、`latency_ms`），但列表接口未关联查询。

## 3. 修复方案

采用**方案 A**（后端子查询一次返回），避免前端多次请求。

### 3.1 修改文件

| 层 | 文件 | 改动 |
|----|------|------|
| 后端 model | `admin-backend/src/features/api/model.rs` | `ApiResponse` 新增 `last_tested_at`/`last_test_status`/`last_test_latency_ms` 可选字段；新增 `LastTestInfo` 结构体 |
| 后端 service | `admin-backend/src/features/api/service.rs` | `list_apis` 增加 `fetch_last_tests_batch`，用 `DISTINCT ON` 批量查询每个 API 最新测试记录 |
| 前端 types | `admin-frontend/src/features/api-registrations/types/index.ts` | `ApiRegistration` 接口新增三个可选字段 |
| 前端组件 | `admin-frontend/src/features/api-registrations/components/ApiList.tsx` | 替换硬编码 `—`，展示状态码、延迟、相对时间 |

### 3.2 关键实现

**后端批量查询**（`admin-backend/src/features/api/service.rs`）：

```rust
async fn fetch_last_tests_batch(
    pool: &PgPool,
    api_ids: &[Uuid],
) -> Result<HashMap<Uuid, LastTestInfo>, AppError> {
    let rows: Vec<LastTestInfo> = sqlx::query_as(
        r#"SELECT DISTINCT ON (api_id)
           api_id, created_at, response_status, latency_ms
           FROM test_histories
           WHERE api_id = ANY($1)
           ORDER BY api_id, created_at DESC"#,
    )
    .bind(api_ids)
    .fetch_all(pool)
    .await
    .unwrap_or_default();
    Ok(rows.into_iter().map(|r| (r.api_id, r)).collect())
}
```

使用 PostgreSQL `DISTINCT ON` 语法，一次查询获取所有 API 的最新测试记录，O(N) 复杂度。

**前端渲染逻辑**（`admin-frontend/src/features/api-registrations/components/ApiList.tsx`）：

- 有测试记录时：显示状态码（2xx 绿色，4xx/5xx 红色）+ 延迟 + 相对时间（hover 显示完整时间）
- 无测试记录时：显示 `—`
- 后端通过 `skip_serializing_if = "Option::is_none"` 确保未测试 API 的响应中不出现 `null` 字段

## 4. 验证结果

- Rust 编译：`cargo check` 通过
- TypeScript 类型检查：`tsc --noEmit` 零错误（涉及文件）
- 向后兼容：`last_test_*` 字段全部可选，`skip_serializing_if` 确保旧客户端不受影响

## 5. 预防建议

- 在实现新页面/表格列时，如果数据源不存在应标记为 TODO 而非硬编码占位符
- 前端表格列应始终绑定数据字段，空值由渲染逻辑处理
