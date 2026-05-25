# Bug: pino `logMethod` hook 未调用 `method()` 导致全部日志静默丢失

| 字段 | 值 |
|------|------|
| ID | 20260525-001 |
| 日期 | 2026-05-25 |
| 严重程度 | **HIGH** — 所有 pino 业务日志完全不可见 |
| 影响范围 | `packages/backend` 全部接口调用过程，包括启动日志、请求处理、AI 调用、错误追踪 |
| 状态 | fixed |
| 分类 | integration |

## 1. 错误现象

后端服务启动后，控制台只显示 Hono 框架的 HTTP 请求/响应日志（`<-- GET /api/...`），所有 pino 业务日志完全不可见：

- 启动日志 `Server running on http://localhost:...` 不显示
- 请求处理日志 `Chat request received`、`Trigger pipeline executing` 不显示
- AI 调用日志 `AI stream request started` 不显示
- 错误日志不显示

## 2. 根因分析

### 2.1 直接原因：`logMethod` hook 未调用 `method()`

**文件**: `packages/backend/src/shared/logger.ts`

pino v10 的 `logMethod` hook 是**替换式 hook** — hook 接收原始日志方法 `method`，必须显式调用它才能输出日志。

```typescript
// BUG 代码 — 只返回 args，从不调用 method
hooks: {
    logMethod(inputArgs: unknown[], method: pino.LogFn): unknown[] {
        if (inputArgs.length >= 2 && typeof inputArgs[0] === "object") {
            inputArgs[0] = redactCredentialFields(inputArgs[0]);
        }
        return inputArgs as never;  // ← method 从未被调用，日志被静默丢弃
    },
},
```

pino 内部处理流程：

1. 用户调用 `logger.info("message")`
2. pino 拦截调用，传入 `(args, originalMethod)` 给 hook
3. **原代码**：hook 修改 args 后直接 return，`originalMethod` 从未被调用
4. 结果：日志数据不写入 stream，控制台无输出，无任何报错

### 2.2 加剧因素：pino-pretty worker thread 在 tsx+ESM 环境不稳定

原始配置同时使用了 `transport: { target: "pino-pretty" }` worker thread 模式。即使 hook 修复后，worker thread 在 tsx watch + ESM 环境下仍可能静默失败，进一步增加日志丢失风险。

## 3. 修复方案

### 3.1 修复 `logMethod` hook（核心修复）

```typescript
hooks: {
    logMethod(inputArgs: unknown[], method: pino.LogFn): unknown[] {
        if (inputArgs.length >= 2 && typeof inputArgs[0] === "object") {
            inputArgs[0] = redactCredentialFields(inputArgs[0]);
        }
        method.apply(this, inputArgs);  // ← 显式调用原始日志方法
        return inputArgs;
    },
},
```

### 3.2 替换 pino-pretty 为自包含 PrettyStream

用 `Writable` 子类实现同步 pretty-print，消除 worker thread 依赖：

- 直接同步写入 `process.stdout`，无 IPC、无缓冲、无静默失败
- 零外部依赖（不依赖 pino-pretty 的模块加载）
- 生产环境仍使用 `pino.destination(1)` 输出 JSON

## 4. 影响的文件

| 文件 | 变更 |
|------|------|
| `packages/backend/src/shared/logger.ts` | 修复 hook + 替换 pino-pretty 为 PrettyStream |

## 5. 验证

修复后控制台正常输出：

```
INFO  02:07:09 Server running on http://localhost:3000
INFO  02:07:09 Initializing PostgreSQL connection {"schema":"xxx","admin":"xxx"}
<-- POST /api/chat
INFO  02:07:09 Chat request received {"conversationId":"xxx","contentPreview":"..."}
INFO  02:07:09 Trigger pipeline executing {"userMessage":"..."}
INFO  02:07:09 AI stream request started {"model":"qwen-plus","messageCount":1}
```

## 6. 教训

> **pino v10 `logMethod` hook 是替换式 hook，必须显式调用 `method()`。** 只返回修改后的 args 而不调用 `method` 会导致所有日志静默丢失，且不会抛出任何错误。升级 pino 主版本时应仔细阅读 hooks API 变更。
