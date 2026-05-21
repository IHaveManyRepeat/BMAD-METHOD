# Validation Plan — UserSettings Not Sent With Chat Request

**Bug ID**: 20260520-002-usersettings-not-sent
**日期**: 2026-05-20

---

## 自动化验证机制

### 1. 单元测试（已实施）

| 测试 | 验证内容 | 文件 |
|------|----------|------|
| `initializeSettings > 并行调用 fetchSettings 和 getPreferences` | 双 API 并行加载 | `settings-store.test.ts` |
| `initializeSettings > 成功时同时设置 responseMode 和 preferences` | 两个状态同时更新 | `settings-store.test.ts` |
| `initializeSettings > API 失败时不覆盖已有状态` | 错误隔离 | `settings-store.test.ts` |
| `preferences > 构建请求时正确映射为 userSettings` | 数据传递完整性 | `chat-store.test.ts` |
| `preferences 为 null 时 userSettings 为 undefined` | 空值处理 | `chat-store.test.ts` |

### 2. 集成测试（建议实施）

| 测试场景 | 步骤 | 预期 |
|----------|------|------|
| 页面加载时初始化两个 store | 挂载 `ChatPage` → 检查 API 调用 | `GET /api/settings` 和 `GET /api/preferences` 都被调用 |
| 完整数据流验证 | 加载页面 → 设置偏好 → 发送消息 | `POST /api/chat` 请求体包含 `userSettings` |
| localStorage 为空时的降级 | 清除 localStorage → 刷新页面 → 发送消息 | 从 API 恢复偏好，请求正常 |

### 3. 静态分析规则（建议实施）

在 ESLint 配置中添加规则，检测 store 中导出的 `initialize*` 函数是否在组件入口被引用：

```javascript
// .eslintrc — 建议规则
{
  "rules": {
    "no-restricted-syntax": [
      "warn",
      {
        "selector": "ExportNamedDeclaration ~ ClassDeclaration",
        "message": "导出的 initialize 函数应在组件 useEffect 中被调用"
      }
    ]
  }
}
```

### 4. 架构约束

在 CLAUDE.md 或 ADR 中注明：
- 每个 Zustand store 导出 `initialize*` 函数时，必须在组件入口（如 `ChatPage.tsx`）的 `useEffect` 中调用
- 新增 store 的初始化函数时，需同时更新组件入口的调用列表
- 修改 store 初始化逻辑后，需运行全量前端测试验证

---

## 5. 错误映射缺陷验证（2026-05-21 新增）

> 针对报告第 10 节分析的错误信息丢失问题

### 5.1 单元测试（建议实施）

| 测试 | 验证内容 | 文件 |
|------|----------|------|
| `mapErrorToMessage` 传入 ApiError 对象时正确映射 code | `ApiError("msg", 429, "RATE_LIMITED")` → `RATE_LIMITED` | `error-messages.test.ts` |
| `mapErrorToMessage` 传入 SSE error 字符串时匹配中文关键词 | `"发送太快"` → `RATE_LIMITED` | `error-messages.test.ts` |
| `sendMessage` catch 块保留结构化错误信息 | `NonRetryableError` → store 保留 code/status | `chat-store.test.ts` |
| SSE error 事件携带 code 时前端正确映射 | `{ type: "error", code: "SKILL_EXECUTION_ERROR" }` → 对应消息 | `chat-store.test.ts` |
| `finalizeAssistantMessage` 不清除 errorMessage | error 后 done → errorMessage 仍存在 | `chat-store.test.ts` |

### 5.2 集成测试（建议实施）

| 测试场景 | 步骤 | 预期 |
|----------|------|------|
| 后端 429 时前端展示"发送太快了" | mock 429 响应 → 发送消息 | ErrorBanner 显示"发送太快了" |
| 后端 5xx 时前端展示"系统开小差了" | mock 500 响应 → 发送消息 | ErrorBanner 显示"系统开小差了" |
| SSE error 事件后 errorMessage 残留 | mock SSE error → SSE done → 检查 store | errorMessage 非空 |
| 后端 SkillExecutionError 时前端展示具体提示 | mock Skill 执行失败 → 发送消息 | ErrorBanner 显示具体消息（非通用 DEFAULT） |

### 5.3 架构约束（补充）

- 后端 SSE `error` 事件必须携带 `code` 字段（与 HTTP 错误响应格式一致）
- 前端 `chat-store` 存储错误时应保留完整 Error 对象，`ChatArea` 映射时优先使用对象
- `mapErrorToMessage()` 的 string 分支应覆盖常见中文错误关键词

---

## 6. AI 意图识别白名单失效验证（2026-05-21 新增）

> 针对报告第 11 节分析的 activeSkillNames 与注册 skill ID 不匹配问题

### 6.1 单元测试（建议实施）

| 测试 | 验证内容 | 文件 |
|------|----------|------|
| `getFilteredSkills` 当 activeSkillNames 包含注册 ID 时正确返回 | 白名单匹配 | `registry.test.ts` |
| `getFilteredSkills` 当 activeSkillNames 为空时返回空数组 | 白名单为空 = 全部不可用 | `registry.test.ts` |
| `getFilteredSkills` 当 activeSkillNames 与注册 ID 不匹配时返回空数组 | 名称不匹配场景 | `registry.test.ts` |
| `AIIntentHandler.handle` 当 tools 为空时返回 `{ matched: false }` | 无工具快速退出 | `ai-intent.test.ts` |
| `loadActiveSkillNames` 返回的名称与硬编码注册 ID 一致 | 端到端名称匹配 | 集成测试 |
| `getFilteredSkills` platform 过滤：`General` 始终通过，`Aigis` 对 `h5` 用户不通过 | 平台过滤逻辑 | `registry.test.ts` |

### 6.2 冒烟测试（建议实施）

| 测试场景 | 步骤 | 预期 |
|----------|------|------|
| 后端启动后 tools 非空 | 启动后端 → `GET /api/health` → 检查日志中 "Active skill filter refreshed" | activeSkills 包含 `view_land_plot` |
| Skill 触发正常 | 发送 "帮我查看地块列表" → 检查 SSE 流 | 包含 `tool_call` 事件 |
| 白名单不匹配时告警 | 设置错误的 activeSkillNames → 发送消息 | 后端日志 WARN："tools.length === 0" |

### 6.3 数据修复验证

| 步骤 | SQL | 预期 |
|------|-----|------|
| 检查 skills 表现有数据 | `SELECT * FROM admin_test.skills ORDER BY skill_name` | 确认重复记录 |
| 清理 deleted 重复记录 | `DELETE FROM admin_test.skills WHERE status='deleted'` | 清理完毕 |
| 修正 skill_name 和 platform | `UPDATE admin_test.skills SET skill_name='view_land_plot', platform_source='General' WHERE status='active'` | 名称与注册 ID 一致 |
| 重启后端验证 | 重启 → 检查日志 | activeSkills = ["view_land_plot"] |

### 6.4 架构约束（补充）

- admin DB `skills.skill_name` 必须与 `SkillRegistry.register()` 中传入的 `skill.id` 完全一致
- `skills.platform_source` 设为 `General` 表示全平台可用，设为特定值时只对匹配的 platform_source 用户可见
- 新增硬编码 skill 时，必须同时在 admin DB 添加对应的 active 记录，否则白名单过滤会将其排除
- `getFilteredSkills()` 在过滤后为空时应输出 WARN 日志
