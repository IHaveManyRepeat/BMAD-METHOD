# Bug: `parseParamsSchema` 未解析 `param_groups` 导致 API 调用时分组参数全部丢失

| 字段 | 值 |
|------|------|
| ID | 20260525-002 |
| 日期 | 2026-05-25 |
| 严重程度 | **HIGH** — param_groups 定义的参数完全不被传递，导致 API 返回错误数据或无数据 |
| 影响范围 | 所有使用 `param_groups` 定义参数的 Skill/API 调用链，当前确认受影响：`get_ai_report` |
| 状态 | fixed |
| 分类 | integration |

## 1. 错误现象

调用 `get_ai_report` 接口时，运行时日志显示请求参数为空：

```json
{
  "apiAlias": "get_ai_report",
  "method": "GET",
  "url": "http://8.140.113.96:12001/ApiGateway/get?powerVersionCode=dailyreport-v1.0&tenantLoginId=agent2",
  "params": {},
  "body": null
}
```

`params` 为空对象，`body` 为 `null`。该接口在 `params_schema` 中通过 `param_groups` 定义了需要传递的参数，但这些参数在运行时完全丢失。

## 2. 根因分析

### 2.1 数据流断层

```
DB params_schema (含 param_groups)
    │
    ▼
parseParamsSchema()              ← 只解析 obj.params，跳过 obj.param_groups
    │ 返回 []
    ▼
buildParamMappings()             ← 无 param_groups 字段的映射条目
    │
    ▼
mapParamsForApi()                ← 返回 {} (空对象)
    │
    ▼
apiRegistry.call()               ← 传入空 params
    │
    ▼
distributeParams() (caller.ts)   ← param_groups 编码/分发逻辑完整，但收到空输入
    │
    ▼
实际请求：params={}, body=null    ← 分组参数全部丢失
```

### 2.2 直接原因：`parseParamsSchema` 只处理 `obj.params`

**文件**: `packages/backend/src/modules/skill/aggregation-config.ts` (原 line 458-476)

原代码只解析 `params_schema.params`（平面参数数组），完全忽略了 `params_schema.param_groups[].fields`（分组参数字段）：

```typescript
// BUG 代码 — 只看 obj.params
function parseParamsSchema(schema: unknown): RawParamDef[] {
    const obj = schema as Record<string, unknown>;
    const paramsArray = obj.params;
    if (!Array.isArray(paramsArray)) return [];  // ← param_groups 场景直接返回 []
    // ...
}
```

当 API 的参数仅定义在 `param_groups` 中（无平面 `params` 数组）时：
1. `parseParamsSchema` 返回 `[]`
2. `buildParamMappings` 无法创建这些字段的 `ParamMapping`
3. `mapParamsForApi` 没有可映射的参数，返回 `{}`
4. API 被调用时携带空参数

### 2.3 加剧因素：Rust admin-backend 已支持 param_groups，Node.js 运行时未同步

commit `010bdcf` 在 Rust admin-backend 中完整实现了 `param_groups` 解析（`param_analyzer.rs` + `repo.rs`），管理端的骨架生成、参数分析均正确工作。但 Node.js 运行时的 `aggregation-config.ts` 未同步更新，形成管理层与运行时的能力断层。

同时，`caller.ts` 的 `distributeParams` 函数（line 65-156）已完整支持 `param_groups` 的编码和分发，但上游 `mapParamsForApi` 提供了空输入，导致下游能力无法发挥。

## 3. 修复方案

### 3.1 核心修复：`parseParamsSchema` 同时解析 `param_groups[].fields`

新增 `parseSingleParam` 辅助函数消除重复逻辑，`parseParamsSchema` 同时处理两个来源：

```typescript
function parseSingleParam(
    item: Record<string, unknown>,
    defaultPosition: string = "query",
): RawParamDef {
    return {
        name: item.name as string,
        paramType: (item.type as string) ?? "string",
        position: (item.in as string) ?? defaultPosition,
        required: (item.required as boolean) ?? false,
    };
}

function parseParamsSchema(schema: unknown): RawParamDef[] {
    const obj = schema as Record<string, unknown>;
    const results: RawParamDef[] = [];

    // 1. Flat params array
    const paramsArray = obj.params;
    if (Array.isArray(paramsArray)) {
        for (const item of paramsArray) {
            if (typeof item === "object" && item !== null && typeof item.name === "string") {
                results.push(parseSingleParam(item));
            }
        }
    }

    // 2. param_groups fields — each field inherits group's position (in)
    const paramGroups = obj.param_groups;
    if (Array.isArray(paramGroups)) {
        for (const group of paramGroups) {
            const g = group as Record<string, unknown>;
            const fields = g.fields;
            const groupPosition = (g.in as string) ?? "query";
            if (!Array.isArray(fields)) continue;

            for (const field of fields) {
                if (typeof field === "object" && field !== null && typeof field.name === "string") {
                    results.push(parseSingleParam(field, groupPosition));
                }
            }
        }
    }

    return results;
}
```

关键设计决策：
- 分组字段继承其所在组的 `in`（位置）属性，默认 `query`
- `parseSingleParam` 复用逻辑，DRY 原则
- 与 `caller.ts` 的 `ParamGroup` 类型定义（`types.ts:107`）保持一致

### 3.2 修复后的完整数据流

```
DB params_schema (含 param_groups)
    │
    ▼
parseParamsSchema()              ← 同时解析 params + param_groups[].fields
    │ 返回完整参数列表
    ▼
buildParamMappings()             ← 为 param_groups 字段创建映射
    │
    ▼
mapParamsForApi()                ← 映射 param_groups 字段到 API 参数
    │ 返回 { field1: val1, field2: val2, ... }
    ▼
apiRegistry.call()               ← 传入完整 params
    │
    ▼
distributeParams() (caller.ts)   ← 收集 group 字段 → base64_json 编码 → 分发
    │
    ▼
实际请求：params={groupName: encodedValue}  ← 分组参数正确传递
```

## 4. 影响的文件

| 文件 | 变更 |
|------|------|
| `packages/backend/src/modules/skill/aggregation-config.ts` | 重写 `parseParamsSchema`，新增 `parseSingleParam`，支持 `param_groups` 解析 |

## 5. 验证

### 5.1 编译验证

修改后 `tsc --noEmit` 无新增错误（所有预存错误均与本次修改无关）。

### 5.2 逻辑验证

修复后的 `parseParamsSchema` 对于以下 `params_schema` 结构：

```json
{
  "params": [...],
  "param_groups": [
    {
      "name": "filterGroup",
      "in": "query",
      "encoding": "base64_json",
      "fields": [
        { "name": "landPlotId", "type": "string", "required": true },
        { "name": "cropName", "type": "string", "required": false }
      ]
    }
  ]
}
```

将正确提取 `landPlotId` 和 `cropName` 为 `RawParamDef`（position 继承组的 `"query"`），进入 `buildParamMappings` → `mapParamsForApi` → `distributeParams` 完整链路。

## 6. 教训

> **跨语言实现的功能必须同步到所有运行时消费者。** 当 admin-backend (Rust) 新增 `param_groups` 解析时，Node.js 运行时的 `aggregation-config.ts` 也应同步更新。`caller.ts` 已有完整的 `param_groups` 编码/分发能力，但因上游参数提取缺失而无法发挥作用——形成"下游能力就绪但上游数据缺失"的隐蔽断层。
>
> 此类 bug 的特征：管理端（skeleton 生成、参数分析）正常，但运行时（实际 API 调用）静默失败，无报错，无异常——仅表现为空参数。
