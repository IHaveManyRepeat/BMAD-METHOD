# Bug Analysis Report

**Bug ID**: 20260520-002-usersettings-not-sent
**日期**: 2026-05-20
**严重级别**: High — 所有依赖用户凭证的 Skill（地块查询、考勤等）无法正常工作
**Bug 类型**: state-management（状态管理问题）
**状态**: 已修复

---

## 1. Bug 描述

### 现象

用户在齿轮设置面板填写了 `cellphone`、`access_token`、`tenant_id` 等参数后，发送聊天消息（如"我有几块地？"），`POST /api/chat` 请求体中 `userSettings` 字段缺失，导致后端 Skill 执行时拿不到用户凭证，第三方 API 调用失败。

### 复现步骤

1. 打开聊天页面
2. 点击齿轮图标，进入设置面板
3. 填写 cellphone、access_token、tenant_id 等字段并保存
4. 刷新页面（或在新浏览器/清缓存后重新打开）
5. 在聊天框输入"我有几块地？"并发送
6. 观察网络请求：`POST /api/chat` 请求体中 `userSettings` 字段为空或缺失

### 预期行为

`POST /api/chat` 请求体包含 `userSettings` 字段，携带用户在设置面板填写的所有偏好值（cellphone、accessToken、tenantId 等），后端 Skill 能正常读取并调用第三方 API。

### 实际行为

`POST /api/chat` 请求体中 `userSettings` 字段缺失（`undefined`），后端 Skill 执行时拿不到用户凭证，第三方 API 调用失败。仅在 localStorage 有缓存时能临时正常工作。

---

## 2. 代码调用图

```
ChatPage.tsx (初始化入口)
  ├─→ initializeStore()         [chat-store]    → 加载会话列表 ✅
  └─→ initializeSettings()      [settings-store] → 加载偏好  ❌ 从未调用

settings-store.initializeSettings()
  ├─→ fetchSettings() (GET /api/settings)     → responseMode
  └─→ getPreferences() (GET /api/preferences) → preferences

用户发送消息:
ChatPage.tsx → chat-store.sendMessage()
  └─→ settings-store.getState().preferences   ← 🔴 此处断裂（preferences 为 null）
        └─→ preferences ?? undefined → undefined
              └─→ api-client → POST /api/chat { body: { userSettings: undefined } }
```

### 🔴 状态初始化断裂

```
初始化链:
  ChatPage useEffect → initializeStore()  ✅ 已调用
  ChatPage useEffect → initializeSettings()  ❌ 缺失调用

结果:
  settings-store.preferences = null (初始值)
  → chat-store 构建 userSettings 时 preferences ?? undefined → undefined
  → POST /api/chat 请求体缺少 userSettings
```

---

## 3. 根因分析

### 3.1 直接原因

`ChatPage.tsx` 页面初始化时只调用了 `chat-store.initializeStore()`，从未调用 `settings-store.initializeSettings()`。用户偏好 (`preferences`) 只能从 localStorage 恢复，如果 localStorage 为空（新浏览器/清缓存/首次访问），`preferences` 就是 `null`，`userSettings` 不会随请求发送。

```typescript
// ChatPage.tsx — 修复前
useEffect(() => {
    initializeStore();      // ✅ 初始化聊天
    // initializeSettings()  ❌ 从未调用
}, [initializeStore]);
```

### 3.2 根本原因（多轮重构遗留）

此 bug 经历了三轮迭代，每轮修复了一个层面但暴露了新层面：

**第一轮（v1.0→v1.1）：** 后端从 DB 按 conversationId 读取用户设置，前端不传凭证。改为前端通过 `settings-store` 管理凭证，`chat-store` 手动映射后发送。但凭证字段硬编码。

**第二轮（v1.2）：** 从硬编码字段迁移到动态键值对（`z.record(z.string(), z.unknown())`），`settings-store` 用 `preferences` 替代 `credentials`。但 `PreferencesForm` 仅在用户点击"保存"时同步到 settings-store，`initializeSettings()` 从未被调用。

**第三轮（本轮）：** 发现 `initializeSettings()` 在生产代码中从未被调用，页面刷新后偏好丢失。此外 `GET /api/settings` 返回未 JSON.parse 的原始字符串，与 `GET /api/preferences` 的行为不一致。

### 3.3 工作流层面根因

| 层级 | 原因 | 位置 |
|------|------|------|
| **L1: 初始化函数未调用** | `initializeSettings()` 有定义、有测试，但从未在组件树中被调用 | `ChatPage.tsx` |
| **L2: 测试覆盖盲区** | 单元测试覆盖了函数逻辑但未覆盖"函数是否在正确的生命周期被调用" | 测试策略 |
| **L3: 重构缺少回归检查** | 从 `credentials` 迁移到 `preferences` 时，未验证初始化流程是否完整保留 | 重构流程 |

---

## 4. 修复方案

### 策略：补全初始化调用 + 并行加载

| 文件 | 变更 | 原则 |
|------|------|------|
| `ChatPage.tsx` | useEffect 中增加 `initializeSettings()` 调用 | 完整初始化：两个 store 都需启动 |
| `settings-store.ts` | `initializeSettings` 改用 `getPreferences()` API + `Promise.allSettled` 并行加载 | DRY: 统一 API 调用 |
| `settings-store.test.ts` | 更新 mock 适配双 API 调用 | 测试与实现同步 |

### 修复后数据流

```
App 启动
  ├─ ChatPage.tsx → initializeStore()         → 加载会话列表
  └─ ChatPage.tsx → initializeSettings()      → 加载 responseMode + preferences
        ├─ fetchSettings() (GET /api/settings)     → responseMode
        └─ getPreferences() (GET /api/preferences) → { cellphone, accessToken, tenantId, ... }

用户发送消息 "我有几块地？"
  ├─ chat-store.ts → preferences = useSettingsStore.getState().preferences  ✅ 从 API 加载
  ├─ api-client.ts → body.userSettings = { cellphone, accessToken, ... }   ✅ 随请求发送
  └─ POST /api/chat → userSettings → extractPlatformSource + validateUserFields + skill.handler
```

---

## 5. 影响范围

### 修改文件

| 文件 | 变更 |
|------|------|
| `packages/frontend/src/chat/components/ChatPage.tsx` | +2 行：调用 `initializeSettings()` |
| `packages/frontend/src/chat/stores/settings-store.ts` | 重构 `initializeSettings`，引入 `getPreferences` + `Promise.allSettled` |
| `packages/frontend/src/chat/stores/settings-store.test.ts` | 更新 mock 适配双 API 调用 |

### 未修改文件

- `chat-store.ts` — 无需修改（已正确读取 `preferences`）
- `api-client.ts` — 无需修改（已正确构建 `userSettings`）
- `PreferencesForm.tsx` — 无需修改（保存逻辑正确）
- 后端 — 无需修改

---

## 6. 验证结果

| 测试套件 | 结果 |
|----------|------|
| settings-store.test.ts | 13/13 通过 |
| chat-store.test.ts | 59/59 通过 |
| SettingsPanel.test.tsx | 10/10 通过 |
| 全部前端测试 | 338/338 通过 |

---

## 7. 附注：Vite Dev Server 转换缓存问题

### 现象

修复代码已正确写入磁盘，TypeScript 编译通过，但浏览器中 `POST /api/chat` 请求体仍缺少 `userSettings`。

### 根因

Vite dev server 将 `.tsx` 转换为 `.js` 时，外部修改（如 AI 编码助手直接写入磁盘）可能未被文件系统监听器检测到，导致继续提供缓存的旧版转换结果。

### 解决方案

重启 Vite dev server，强制重新转换所有模块。

### 教训

**Vite HMR 不一定可靠：** 外部工具修改源文件时，**重启 dev server 是最可靠的验证手段**。"源码正确"和"运行时加载的是新代码"是两件事。

---
---

## 8. 实际修复方案补充（2026-05-21）

> 上述第 4 节的分析仅覆盖了"前端初始化未调用"这一层面。实际修复发现，问题远不止此——整个 `credentials` → `preferences` 的迁移涉及前后端全链路重构。以下是实际执行的完整修复方案。

### 8.1 问题全貌

原始分析认为只需在 `ChatPage` 补全 `initializeSettings()` 调用，但实际代码存在更深层的架构问题：

| 层面 | 原始分析 | 实际情况 |
|------|---------|---------|
| 前端初始化 | `initializeSettings()` 未被调用 ✅ 正确 | — |
| 前端数据模型 | 认为无需修改 `chat-store`、`api-client` | `chat-store` 仍使用 `UserCredentialsPayload`（硬编码字段），需改为动态 `UserSettingsPayload` |
| 后端 Schema | 认为"后端无需修改" | 后端仍使用固定字段 `UserSettingsPayloadSchema`，且依赖 `UserSettingService` 做 DB 持久化 |
| Skill 加载 | 未涉及 | `SkillLoader` 尚不支持从 admin DB 动态 import，`preferenceInjector` 仍在代码中 |
| 删除代码 | 未涉及 | `preferenceInjector.ts`（376 行）及其测试（607 行）需删除 |

### 8.2 实际修复内容

#### 8.2.1 前端变更

**ChatPage.tsx** — 补全初始化调用（与原分析一致）

```typescript
// 修复后
useEffect(() => {
    initializeStore();       // ✅ 加载会话列表
    initializeSettings();    // ✅ 加载 responseMode + preferences
}, [initializeStore, initializeSettings]);
```

**chat-store.ts** — 动态键值对替代硬编码映射（原分析遗漏）

```typescript
// 修复前：硬编码字段映射
const credentials = useSettingsStore.getState().credentials;
const userSettings: UserCredentialsPayload | undefined = credentials
    ? {
        cellphone: credentials.cellphone,
        accessToken: credentials.accessToken,
        platformSource: credentials.platformSource,
        tenantId: credentials.tenantId,
      }
    : undefined;

// 修复后：直接透传动态键值对
const preferences = useSettingsStore.getState().preferences;
const userSettings: UserSettingsPayload | undefined = preferences ?? undefined;
```

**api-client.ts** — 接口类型动态化（原分析遗漏）

```typescript
// 修复前：固定字段接口
export interface UserCredentialsPayload {
    readonly cellphone?: string;
    readonly accessToken?: string;
    readonly platformSource?: string;
    readonly tenantId: string;
    readonly areaCode?: string;
    readonly longitude?: string;
    readonly latitude?: string;
}

// 修复后：动态键值对
export type UserSettingsPayload = Record<string, unknown>;
```

**settings-store.ts** — 并行加载 API 数据（原分析已覆盖，实际实现一致）

使用 `Promise.allSettled` 并行调用 `fetchSettings()` 和 `getPreferences()`，过滤空值后存入 `preferences`。

#### 8.2.2 后端变更（原分析完全未涉及）

**chat.ts** — 移除硬编码 Schema 和 DB 持久化

```typescript
// 修复前：固定字段 Zod schema + UserSettingService 依赖
const UserSettingsPayloadSchema = z.object({
    cellphone: z.string().min(1).optional(),
    accessToken: z.string().min(1).optional(),
    platformSource: z.enum(PLATFORM_SOURCES).optional(),
    tenantId: z.string().min(1),
    // ...
});
// + extractFieldMap() 从 DB 读取字段映射
// + userSettingService.updateSettings() 持久化到 DB

// 修复后：动态键值对，无 DB 持久化
const SendChatSchema = z.object({
    conversationId: z.string().min(1),
    content: z.string().min(1).max(50000),
    userSettings: z.record(z.string(), z.unknown()).optional(),
});
```

关键简化：
- 移除 `UserSettingService` 依赖 — 前端每次请求携带完整 userSettings
- 移除 `extractFieldMap()` — 直接使用请求中的 `userSettings`
- 新增 `extractPlatformSource()` — 从 `userSettings.platform_source` 安全提取平台标识
- `userSettings` 透传至 `executeTriggerMatchedSkill()` 和 `executeAIToolCallLoop()`

**skill registry** — 动态技能注册

```typescript
// 新增：动态技能 ID 集合，与硬编码技能隔离
private readonly dynamicSkillIds = new Set<string>();

registerDynamic(skill: SkillDefinition): void {
    this.skills.delete(skill.id);  // 覆盖旧版本，不抛重复异常
    this.skills.set(skill.id, skill);
    this.dynamicSkillIds.add(skill.id);
}

clearDynamicSkills(): void {
    // 只清除动态技能，保护 clock_record 等硬编码技能
    for (const id of this.dynamicSkillIds) {
        this.skills.delete(id);
        this.validationStatus.delete(id);
    }
    this.dynamicSkillIds.clear();
}
```

**skill loader** — 从 admin DB 动态 import

```typescript
class SkillLoader {
    async reloadFromDB(): Promise<void> {
        // 1. 查询 admin DB 中 active skills 的 implementation_path
        // 2. 动态 import('../../{implementation_path}')
        // 3. 调用 module.createSkill(deps)
        // 4. 原子替换：clearDynamicSkills() → registerDynamic()
    }
}
```

**删除 preferenceInjector**

| 文件 | 行数 | 删除原因 |
|------|------|---------|
| `preferenceInjector.ts` | 376 行 | 职责被前端 `userSettings` 直传替代 |
| `preferenceInjector.test.ts` | 607 行 | 随实现删除 |

### 8.3 完整数据流（修复后）

```
App 启动
  ├─ ChatPage.tsx → initializeStore()         → 加载会话列表
  └─ ChatPage.tsx → initializeSettings()      → 加载 responseMode + preferences
        ├─ fetchSettings() (GET /api/settings)     → responseMode
        └─ getPreferences() (GET /api/preferences) → { cellphone, accessToken, tenantId, ... }

用户发送消息 "我有几块地？"
  ├─ chat-store.ts → preferences = useSettingsStore.getState().preferences  ✅
  ├─ api-client.ts → body.userSettings = { cellphone, accessToken, ... }   ✅
  └─ POST /api/chat { userSettings }
        ├─ extractPlatformSource(userSettings)           → "Aigis"
        ├─ triggerPipeline.execute({ platformSource })   → 路由到对应 Skill
        ├─ validateUserFields(skill, userSettings)       → 提取 requiredUserFields
        └─ skill.handler(input, { ...userFields })       → 调用第三方 API ✅
```

### 8.4 实际影响范围（修正第 5 节）

| 文件 | 变更类型 |
|------|---------|
| `packages/frontend/src/chat/components/ChatPage.tsx` | 修改：补全 `initializeSettings()` 调用 |
| `packages/frontend/src/chat/stores/chat-store.ts` | 修改：`credentials` → `preferences`，`UserCredentialsPayload` → `UserSettingsPayload` |
| `packages/frontend/src/chat/stores/settings-store.ts` | 修改：`credentials` → `preferences`，`saveCredentials` → `savePreferences` |
| `packages/frontend/src/chat/stores/settings-store.test.ts` | 重写：适配动态 `preferences` |
| `packages/frontend/src/shared/lib/api-client.ts` | 修改：`UserCredentialsPayload` → `UserSettingsPayload` |
| `packages/backend/src/routes/chat.ts` | 修改：移除 `UserSettingService`、硬编码 Schema、`extractFieldMap()` |
| `packages/backend/src/routes/chat.test.ts` | 修改：适配移除 `UserSettingService` |
| `packages/backend/src/modules/skill/registry.ts` | 修改：新增 `registerDynamic`、`clearDynamicSkills` |
| `packages/backend/src/modules/skill/loader.ts` | 重写：从 admin DB 动态 import 技能 |
| `packages/backend/src/modules/skill/index.ts` | 修改：移除 preferenceInjector 导出 |
| `packages/backend/src/bootstrap.ts` | 修改：接入 `SkillLoader`，LISTEN/NOTIFY 触发重载 |
| `packages/backend/src/modules/chat/trigger/ai-intent.ts` | 修改：适配 `platformSource` 参数 |
| `packages/backend/src/modules/chat/trigger/types.ts` | 修改：`TriggerInput` 新增 `platformSource` |
| `packages/backend/src/modules/skill/preferenceInjector.ts` | **删除**（376 行） |
| `packages/backend/src/modules/skill/preferenceInjector.test.ts` | **删除**（607 行） |

### 8.5 架构决策记录

**ADR-001: 前端直传 userSettings vs 后端 DB 持久化**

决策：前端每次请求携带完整 `userSettings`，后端不持久化。理由：v1.1 的 `UserSettingService` 增加了不必要的 DB 依赖；前端已管理偏好状态，请求携带是自然的数据流。

**ADR-002: 动态键值对 vs 固定 Schema**

决策：统一使用 `Record<string, unknown>`。理由：管理端 `system_context_fields` 定义字段，可动态增减；硬编码字段导致每次改字段都要改前后端。

**ADR-003: SkillLoader 动态 import vs 静态注册**

决策：从 admin DB 读取 `implementation_path`，动态 `import()` 加载。理由：新增/修改技能无需重启后端；`registerDynamic()` 支持覆盖注册，`clearDynamicSkills()` 保护硬编码技能。

---
---

## 9. 全链路数据流验证与残余风险分析（2026-05-21）

> 对修复后的代码进行全链路追踪，验证 userSettings 从前端到 Skill 执行的完整传递路径，并识别残余风险和测试覆盖盲区。

### 9.1 全链路数据流追踪

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 前端                                                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. ChatPage.tsx 初始化                                                   │
│     useEffect → initializeStore()      ✅ 加载会话列表                     │
│     useEffect → initializeSettings()   ✅ 加载 responseMode + preferences │
│                                                                           │
│  2. settings-store.initializeSettings()                                  │
│     ├─ fetchSettings()     GET /api/settings     → responseMode           │
│     └─ getPreferences()    GET /api/preferences  → preferences            │
│        └─ 过滤: 排除 response_mode、undefined、null、空字符串              │
│        └─ 非空 → set({ preferences }) + persistPreferencesToStorage()     │
│                                                                           │
│  3. 用户发送消息 → chat-store.sendMessage()                               │
│     ├─ preferences = useSettingsStore.getState().preferences              │
│     ├─ userSettings = preferences ?? undefined                            │
│     └─ sendChatSSE(convId, content, onEvent, { userSettings })            │
│                                                                           │
│  4. api-client.sendChatSSE()                                             │
│     ├─ body = { conversationId, content }                                 │
│     ├─ if (options.userSettings) body.userSettings = options.userSettings │
│     └─ POST /api/chat  ← 最终 HTTP 请求                                  │
│                                                                           │
├─────────────────────────────────────────────────────────────────────────┤
│ 后端                                                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  5. routes/chat.ts → POST /api/chat                                      │
│     ├─ SendChatSchema.safeParse(body)                                     │
│     │  └─ userSettings: z.record(z.string(), z.unknown()).optional()     │
│     ├─ extractPlatformSource(userSettings)                                │
│     │  └─ userSettings.platform_source → PlatformSource | "General"      │
│     └─ SSE stream:                                                        │
│        ├─ triggerPipeline.execute({ platformSource })                     │
│        │  → AI 意图识别（带 platform 过滤的 tool definitions）              │
│        │                                                                  │
│        ├─ [命中] executeTriggerMatchedSkill(skill, ..., userSettings)     │
│        │  ├─ validateUserFields(skill, userSettings)                      │
│        │  │  └─ skill.requiredUserFields.safeParse(userSettings)          │
│        │  │     → { ok: true, data } 扁平展开注入 ctx                     │
│        │  │     → { ok: false, missingFields, message } → validation_error│
│        │  └─ skill.handler(input, { conversationId, userMessage, ...fields})│
│        │                                                                  │
│        └─ [未命中] executeAIToolCallLoop(..., platformSource, userSettings)│
│           ├─ toFilteredToolDefinitions(platformSource)                     │
│           └─ for each tool_call:                                          │
│              ├─ validateUserFields(skill, userSettings)                    │
│              └─ skill.handler(validatedInput, { ...userFieldsResult.data })│
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

**结论：全链路 userSettings 传递路径完整，无断裂点。**

### 9.2 残余风险

#### 风险 R1: `skill-validator.ts` 中 `FIELD_LABELS` 硬编码

**位置**: `packages/backend/src/modules/skill/skill-validator.ts:83-88`

```typescript
const FIELD_LABELS: Record<string, string> = {
    cellphone: "手机号",
    accessToken: "访问令牌",
    tenantId: "租户ID",
};
```

**问题**: 字段标签硬编码，与管理端 `system_context_fields` 的动态字段定义脱节。新增字段（如 `employee_id`）的校验失败消息会显示原始字段名而非中文标签。

**影响**: 用户友好的校验提示只覆盖了 3 个已知字段。管理端新增任何字段都需要同步修改此代码。

**建议**: 从管理端 API 拉取字段定义（含 description），或让 Skill 定义时在 `requiredUserFields` 的 Zod schema 中用 `.describe()` 提供标签。

#### 风险 R2: `ChatPage.test.tsx` 未验证 `initializeSettings()` 调用

**位置**: `packages/frontend/src/chat/components/ChatPage.test.tsx`

**问题**: 所有 5 个测试用例仅 mock 了 `GET /api/conversations` 和 `POST /api/conversations`，完全没有 mock 或断言 `GET /api/settings` 和 `GET /api/preferences` 的调用。这是导致原始 bug 的同一类测试盲区——**单元测试覆盖了函数逻辑，但未覆盖函数是否在正确的生命周期被调用**。

**具体缺失**:
- 无测试验证 ChatPage 挂载时触发了 `GET /api/settings`
- 无测试验证 ChatPage 挂载时触发了 `GET /api/preferences`
- 无测试验证 `initializeSettings()` 完成后 `preferences` 被正确设置

**影响**: 如果未来重构再次遗漏 `initializeSettings()` 调用，现有测试无法检测到回归。

**建议**: 在 `ChatPage.test.tsx` 中增加集成测试：
```typescript
it("should call initializeSettings on mount", async () => {
    const fetchMock = vi.fn()
        .mockResolvedValueOnce({ /* conversations */ })
        .mockResolvedValueOnce({ /* create conversation */ })
        .mockResolvedValueOnce({ /* GET /api/settings */ })
        .mockResolvedValueOnce({ ok: true, status: 200, json: () => Promise.resolve({ ok: true, data: { cellphone: "138..." }}) });

    vi.stubGlobal("fetch", fetchMock);
    render(<ChatPage />);

    await waitFor(() => {
        // 验证 fetch 被调用了 4 次（含 settings + preferences）
        expect(fetchMock).toHaveBeenCalledTimes(4);
    });
});
```

#### 风险 R3: `initializeSettings()` 空响应不覆盖 localStorage 旧数据

**位置**: `packages/frontend/src/chat/stores/settings-store.ts:95-98`

```typescript
if (Object.keys(prefs).length > 0) {
    set({ preferences: prefs });
    persistPreferencesToStorage(prefs);
}
```

**问题**: 如果 API 返回的所有偏好值都被过滤掉（空字符串、undefined、null 或仅有 `response_mode`），`preferences` 不会被更新。这意味着：
- 用户在浏览器 A 清除了设置 → 浏览器 B 的 localStorage 仍保留旧数据
- API 返回空偏好时，localStorage 中的过期数据不会被清除

**影响**: 低概率场景，但会导致用户认为设置已清除而实际未清除的不一致。

#### 风险 R4: Preferences Schema 缓存 TTL 5 分钟

**位置**: `packages/backend/src/routes/preferences.ts:39`

```typescript
const SCHEMA_CACHE_TTL_MS = 5 * 60 * 1000; // 5 minutes
```

**问题**: 管理端新增 `system_context_fields` 字段后，用户端 `GET /api/preferences/schema` 最多 5 分钟后才返回新字段定义。在此窗口期内：
- `PreferencesForm` 不渲染新字段（schema 缓存了旧版本）
- `PUT /api/preferences` 校验新字段会返回 "未知的偏好字段" 错误

**影响**: 管理端变更字段后的 5 分钟内用户无法使用新字段。属于可接受的传播延迟，但应在管理端操作说明中注明。

#### 风险 R5: `settings-store` 初始化与 `sendMessage` 的竞态

**问题**: `ChatPage` 的 `useEffect` 中 `initializeStore()` 和 `initializeSettings()` 是并行发起的。如果用户在 `initializeSettings()` 完成**之前**快速发送消息，`preferences` 可能仍是 `null`（或 localStorage 中的旧值）。

**时序**:
```
t0: useEffect → initializeStore() + initializeSettings() 同时发起
t1: initializeStore() 完成，会话加载完毕，用户看到聊天界面
t2: 用户快速发送消息
t3: sendMessage() → preferences = null → userSettings = undefined
t4: initializeSettings() 完成，preferences 被设置
```

**影响**: 极端场景——用户需要在页面加载后 1-2 秒内就发送消息且 API 响应较慢时才会触发。`settings-store` 的初始值已从 localStorage 恢复，所以只要 localStorage 非空就不会丢失。

### 9.3 测试覆盖矩阵

| 链路节点 | 单元测试 | 集成测试 | 覆盖状态 |
|----------|---------|---------|---------|
| `ChatPage` 调用 `initializeSettings()` | — | ❌ 无断言 | **盲区** |
| `settings-store.initializeSettings()` 逻辑 | ✅ 5 个测试 | — | 覆盖 |
| `chat-store.sendMessage()` 读取 preferences | ✅ 3 个测试 | — | 覆盖 |
| `sendChatSSE()` 构建请求体 | ✅ 随 chat-store 测试覆盖 | — | 覆盖 |
| 后端 `SendChatSchema` 校验 | ✅ chat.test.ts 覆盖 | — | 覆盖 |
| `extractPlatformSource()` | ✅ 随集成路径覆盖 | — | 覆盖 |
| `validateUserFields()` | ✅ 6 个测试 | — | 覆盖 |
| `skill.handler()` 接收 userFields | ✅ 随各 Skill 测试覆盖 | — | 覆盖 |
| **完整 E2E**: ChatPage 挂载 → 设置偏好 → 发消息 → 后端收到 userSettings | — | ❌ 无 | **盲区** |

### 9.4 根因模式总结

此 bug 属于**初始化遗漏**类问题，具有以下可识别的模式特征：

1. **跨 Store 调用链断裂**: `ChatPage` 需要初始化两个独立的 Zustand store（`chat-store` + `settings-store`），但只有一个被调用。这类问题在 Zustand 架构中容易发生——每个 store 独立定义 `initialize*` 函数，但没有强制机制确保组件入口调用所有初始化。

2. **测试覆盖结构性盲区**: 单元测试覆盖了每个 store 的初始化函数逻辑，但没有任何测试验证"页面挂载时是否调用了所有必要的初始化函数"。这是经典的**集成测试缺失**——各组件单独正确，但组装后的行为未验证。

3. **多轮迭代的知识债务**: 从 `credentials`（v1.0）→ `UserCredentialsPayload`（v1.1）→ `preferences` + `UserSettingsPayload`（v1.2）的迁移过程中，每轮修复者都基于上一轮的理解工作，但上下文传递链越来越长，没有人做全链路验证。

### 9.5 防复发建议（补充）

基于残余风险分析，补充以下防复发措施：

| 编号 | 措施 | 针对风险 | 优先级 |
|------|------|---------|--------|
| D1 | `ChatPage.test.tsx` 增加挂载时 `initializeSettings()` 调用验证 | R2 | **P0** |
| D2 | `skill-validator.ts` 的 `FIELD_LABELS` 改为从管理端 schema 动态获取或用 Zod `.describe()` | R1 | P1 |
| D3 | `initializeSettings()` API 返回空偏好时，清除 localStorage 中的旧数据 | R3 | P2 |
| D4 | 在管理端字段变更操作说明中注明 5 分钟缓存延迟 | R4 | P2 |
| D5 | 考虑 `sendMessage()` 中等待 `settings-store.initialized === true` 后再读取 preferences | R5 | P3 |

---
---

## 10. 新增分析：Chat 请求后前端显示"出了点问题"通用错误（2026-05-21）

> 用户报告：chat 接口请求后，页面报错"出了点问题，刷新页面试试，你刚才的内容不会丢"。
> 本节分析该通用错误消息的触发路径和根因模式。

### 10.1 错误消息来源追踪

用户看到的"出了点问题，刷新页面试试，你刚才的内容不会丢"来自 `error-messages.ts` 的 `DEFAULT_ERROR` 常量（第 28-32 行）：

```typescript
const DEFAULT_ERROR: ErrorMessage = {
    title: "出了点问题",
    description: "刷新页面试试，你刚才的内容不会丢",
    variant: "generic",
};
```

该常量仅在 `mapErrorToMessage()` 函数无法匹配任何已知错误模式时返回。

### 10.2 错误触发的两条路径

**路径 A：HTTP 请求阶段异常（`sendChatSSE` throw）**

```
chat-store.sendMessage() → sendChatSSE() → fetch POST /api/chat
  ├─ 4xx → NonRetryableError(message, status, code) → catch 块
  ├─ 5xx → ApiError("服务器暂时不可用", status, "SERVER_ERROR") → catch 块
  └─ retry 耗尽 → ApiError → catch 块

catch 块 (chat-store.ts:258-270):
  const message = error instanceof Error ? error.message : "发送消息失败"
  → set({ errorMessage: message })
  → ChatArea.tsx: mapErrorToMessage(errorMessage) // 传入 string
```

**路径 B：SSE 流中后端发送 `error` 事件**

```
后端 chat.ts → createSSEStream → try { await onStream(emit) }
  catch (error) {
    emit({ type: "error", message: error.message })
    emit({ type: "done", conversationId: "" })
  }

前端 chat-store.ts:225-227:
  event.type === "error" → setError(data.message) → set({ errorMessage: message })
  → ChatArea.tsx: mapErrorToMessage(errorMessage) // 传入 string
```

### 10.3 根因：错误信息丢失结构化 Code

`chat-store.sendMessage()` 的 catch 块将 `ApiError`（含 `code`、`status`、`message`）降级为纯字符串：

```typescript
// chat-store.ts:259 — 只取 message 字符串，丢弃 code 和 status
const message = error instanceof Error ? error.message : "发送消息失败";
set({ errorMessage: message });
```

然后 `ChatArea.tsx:146` 调用：

```typescript
mapErrorToMessage(errorMessage) // 参数是 string，不是 Error 对象
```

在 `mapErrorToMessage()` 中，string 输入只做关键词匹配（第 82-91 行）：

```typescript
if (typeof error === "string") {
    const lower = error.toLowerCase();
    if (lower.includes("network") || lower.includes("fetch") || lower.includes("连接"))
        return NETWORK_ERROR;
    if (lower.includes("timeout") || lower.includes("超时") || lower.includes("abort"))
        return TIMEOUT_ERROR;
    return DEFAULT_ERROR;  // ← 所有中文错误消息都会走到这里
}
```

**核心问题：** 后端错误消息是中文（如"查询出了点问题，稍后再试试"、"服务器暂时不可用"），而 `mapErrorToMessage()` 的 string 分支只匹配英文关键词 `network`/`fetch`/`timeout`/`abort`。即使是中文的"超时"能匹配，但"连接"也只在特定场景下出现。大部分后端错误消息都命中 `DEFAULT_ERROR`。

### 10.4 SSE error + done 双事件的副作用

后端 `createSSEStream` 在 catch 中依次 emit `error` 和 `done`（emitter.ts:24-28）：

```typescript
catch (error) {
    emit({ type: "error", message });
    emit({ type: "done", conversationId: "" });
}
```

前端处理顺序：
1. `error` 事件 → `setError(message)` → `{ errorMessage: msg, status: "error" }`
2. `done` 事件 → `finalizeAssistantMessage()` → `{ status: "idle" }`（**不清除 errorMessage**）

结果：`errorMessage` 残留在 store 中，即使状态已恢复 idle，ErrorBanner 仍然显示。下次发送消息时 `setInputText` 检测到 `errorMessage` 存在才会清除。

### 10.5 后端 TypeScript 编译错误（潜在运行时风险）

当前后端存在编译错误：

| 文件 | 错误 | 运行时影响 |
|------|------|-----------|
| `plot_list.ts:99` | `"invalid_input"` 不在 `SkillErrorCategory` 中（应为 `"unknown"`） | 类型擦除后运行时不报错，但 category 语义错误 |
| `aggregation-config.ts:195` | `sql.identifier()` 返回值类型不匹配 `SQL` | 运行时可能正常（drizzle-orm 推断问题），但不保证 |

### 10.6 完整错误映射缺陷矩阵

| 后端错误场景 | 后端错误消息 | `mapErrorToMessage()` 映射结果 | 问题 |
|-------------|------------|------------------------------|------|
| AI Provider 超时 | "AI 服务暂时不可用" | `DEFAULT_ERROR` | 应映射为 `TIMEOUT_ERROR` |
| Skill 执行失败 | "查询出了点问题，稍后再试试" | `DEFAULT_ERROR` | 无对应映射 |
| API 429 Rate Limit | "发送消息太频繁，请稍后再试" | `DEFAULT_ERROR` | 应映射为 `RATE_LIMITED` |
| API 404 Not Found | "会话不存在" | `DEFAULT_ERROR` | 应映射为 `NOT_FOUND` |
| API 5xx | "服务器暂时不可用" | `DEFAULT_ERROR` | 应映射为 `SERVER_ERROR` |
| 网络错误 | "Failed to fetch" | `NETWORK_ERROR` | 唯一正确映射的场景 |
| SSE 流为空 | "响应流为空" | `DEFAULT_ERROR` | 应映射为 `NETWORK_ERROR` |

### 10.7 修复方案

**方案 A（推荐）：保留结构化错误信息**

修改 `chat-store.sendMessage()` 的 catch 块，将完整 `ApiError` 对象（含 code/status）存储到 store，而非仅存 message 字符串：

```typescript
// chat-store.ts — catch 块修改
catch (error) {
    // 保留结构化错误信息给 mapErrorToMessage 使用
    set((state) => ({
        status: "error" as const,
        errorObject: error,       // 新增：保留完整 Error 对象
        errorMessage: error instanceof Error ? error.message : "发送消息失败",
        inputText: userContent,
        messages: { ...state.messages, [activeConversationId]: filtered },
    }));
}
```

修改 `ChatArea.tsx` 优先使用 `errorObject`：

```typescript
{errorMessage && (
    <ErrorBanner
        error={errorObject ? mapErrorToMessage(errorObject) : mapErrorToMessage(errorMessage)}
    />
)}
```

**方案 B（快速修复）：扩展 `mapErrorToMessage` 的 string 匹配**

在 string 分支增加中文关键词匹配：

```typescript
if (typeof error === "string") {
    const lower = error.toLowerCase();
    if (lower.includes("network") || lower.includes("fetch") || lower.includes("连接") || lower.includes("网络"))
        return NETWORK_ERROR;
    if (lower.includes("timeout") || lower.includes("超时") || lower.includes("abort") || lower.includes("太久"))
        return TIMEOUT_ERROR;
    if (lower.includes("频繁") || lower.includes("太快"))
        return mapErrorCodeToMessage("RATE_LIMITED");
    if (lower.includes("不存在") || lower.includes("找不到"))
        return mapErrorCodeToMessage("NOT_FOUND");
    return DEFAULT_ERROR;
}
```

**方案 C：SSE error 事件携带结构化 code**

修改后端 `createSSEStream` 在 error 事件中附带 code：

```typescript
// emitter.ts
catch (error) {
    const code = error instanceof AppError ? error.code : "INTERNAL_ERROR";
    const message = error instanceof Error ? error.message : "AI 服务暂时不可用";
    emit({ type: "error", message, code });  // 新增 code
}
```

前端 chat-store 在 SSE error 处理中使用 code：

```typescript
} else if (event.type === "error") {
    const code = data.code as string | undefined;
    const message = (data.message as string) ?? "未知错误";
    // 使用 code 优先映射
    get().setError(code ? mapErrorCodeToMessage(code) : message);
}
```

### 10.8 修复后错误路径验证矩阵

| 后端错误场景 | 修复后映射路径 | 预期前端展示 |
|-------------|-------------|------------|
| AI Provider 超时 | SSE error → code: "INTERNAL_ERROR" → `SERVER_ERROR` | "系统开小差了" |
| Skill 执行失败（SkillExecutionError） | SSE error → code: "SKILL_EXECUTION_ERROR" → `DEFAULT_ERROR`（可新增映射） | "出了点问题"（可优化为具体提示） |
| API 429 | HTTP 429 → ApiError(code:"RATE_LIMITED") → `RATE_LIMITED` | "发送太快了" |
| API 404 | HTTP 404 → ApiError(code:"NOT_FOUND") → `NOT_FOUND` | "找不到内容" |
| API 5xx | HTTP 5xx → ApiError(code:"SERVER_ERROR") → `SERVER_ERROR` | "系统开小差了" |

### 10.9 风险评估

| 风险 | 级别 | 说明 |
|------|------|------|
| 用户无法区分错误类型 | **High** | 所有错误显示同一个通用消息，用户不知道是网络问题、凭证过期还是服务故障 |
| 诊断困难 | **Medium** | 前端只记录通用错误消息，无法通过用户反馈定位具体后端问题 |
| `errorMessage` 残留 | **Low** | `finalizeAssistantMessage` 不清除 `errorMessage`，但 `setInputText` 会在下次输入时清除 |
| SSE error 事件 `done` 后 status 不一致 | **Low** | `error` 设 `status: "error"`，`done` 设 `status: "idle"`，`errorMessage` 仍在 |

### 10.10 防复发建议（补充）

| 编号 | 措施 | 针对风险 | 优先级 |
|------|------|---------|--------|
| D6 | `chat-store` 存储完整 Error 对象，`ChatArea` 优先用结构化映射 | 错误分类丢失 | **P0** |
| D7 | SSE error 事件附带 `code` 字段，前端用 `code` 做 mapErrorCodeToMessage | SSE 路径无 code | **P0** |
| D8 | `finalizeAssistantMessage` 检测 `errorMessage` 存在时不覆盖 `status`，或在 `done` 处理中清除 errorMessage | errorMessage 残留 | P1 |
| D9 | `mapErrorToMessage` string 分支增加中文关键词覆盖（"频繁"、"太久"、"网络"等） | 兜底匹配不足 | P1 |
| D10 | 后端 `plot_list.ts:99` 的 `"invalid_input"` 改为 `"unknown"` | TypeScript 编译错误 | P2 |


---

## 11. 后端根因分析：AI 意图识别 100% 失败（2026-05-21 新增）

> 用户反馈："chat 接口请求后，页面报错出了点问题。刷新页面试试，你刚才的内容不会丢"
> 前端显示的是通用 DEFAULT_ERROR（"出了点问题"），但实际原因是后端 AI 无法识别用户意图。

### 11.1 现象

- 所有聊天请求（无论正向/反向）均无法触发 Skill
- SSE 流只返回 `token` 和 `done` 事件，没有 `tool_call`、`validation_error` 等事件
- Qwen AI 返回的文字回复为通用知识回答（如建议去"不动产登记中心"查地块）
- E2E 意图识别测试：117/117 全部失败（13 个测试文件）

### 11.2 排除的假设

| 假设 | 验证结果 | 结论 |
|------|----------|------|
| Qwen 不支持 `z.toJSONSchema()` 输出的 `$schema`/`additionalProperties` | 直接 SDK 测试：两种格式 Qwen 都能正常 tool_call | ❌ 排除 |
| `INTENT_SYSTEM_PROMPT` 导致 Qwen 不调工具 | 直接用相同 prompt 测试：tool_call 正常 | ❌ 排除 |
| curl 在 Windows bash 中传递中文导致编码问题 | Unicode 转义测试同样失败 | ❌ 排除 |
| Qwen 模型版本不支持 function calling | 直接 SDK 调用完全正常 | ❌ 排除 |

### 11.3 根因

**`activeSkillNames` 白名单过滤导致所有工具定义被清空。**

调用链分析：

```
chat.ts:127  triggerPipeline.execute(input)
  → ai-intent.ts:16  tools = skillRegistry.toFilteredToolDefinitions(input.platformSource)
    → registry.ts:145-153  getFilteredSkills(platformSource)
      → registry.ts:124  if (!this.activeSkillNames.has(skill.id)) return false  ← 💥 过滤掉了所有技能
```

**数据库状态（admin_test.skills 表）：**

| skill_name | status | platform_source | implementation_path |
|------------|--------|-----------------|---------------------|
| plot_list | deleted | General | modules/domain/plot_list/plot_list.ts |
| plot_list | deleted | General | modules/domain/plot_list/plot_list.ts |
| plot_list | **active** | **Aigis** | modules/domain/plot_list/plot_list.ts |

**bootstrap.ts 注册的 skill ID：**
- `view_land_plot`（第 96-97 行 `createViewLandPlotSkill()`）
- `web_search`、`clock_record`、`custom_app_list`、`custom_app_detail`、`agriculture_report`

**矛盾点：**
1. DB 中 `active` 的 `skill_name` 是 `plot_list`，但硬编码注册的 ID 是 `view_land_plot` → **名称不匹配**
2. `platform_source = 'Aigis'`，但用户发送的是 `platform_source = 'h5'` → **平台不匹配**
3. DB 中有两条 `deleted` 的重复记录 → **数据质量问题**

**结果：**
- `loadActiveSkillNames()` 查询 → `Set(["plot_list"])`
- 硬编码注册的 `view_land_plot` 的 `id` 不在白名单中
- `toFilteredToolDefinitions()` 返回 `[]` 空数组
- `AIIntentHandler.handle()` 第 17 行：`tools.length === 0` → 直接返回 `{ matched: false }`
- `executeAIToolCallLoop()` 同样拿到空数组 → Qwen 无工具可用 → 纯文字回复

### 11.4 代码位置

| 文件 | 行号 | 关键逻辑 |
|------|------|---------|
| `bootstrap.ts:70-74` | `reloadActiveSkills()` | 从 admin.skills 查询 `status='active'` 的 skill_name |
| `bootstrap.ts:96-97` | `skillRegistry.register(viewLandPlotSkill)` | 硬编码注册的 skill ID 为 `view_land_plot` |
| `bootstrap.ts:164-165` | `await reloadActiveSkills()` | 初始化 active 白名单 |
| `registry.ts:114-129` | `getFilteredSkills()` | 四重过滤，第 124 行检查 activeSkillNames |
| `registry.ts:145-153` | `toFilteredToolDefinitions()` | 基于 getFilteredSkills 生成 tool 定义 |
| `ai-intent.ts:16-19` | `tools.length === 0` check | 无工具直接返回 unmatched |

### 11.5 影响范围

- **所有 Skill 不可用**：不仅是 `view_land_plot`，所有硬编码注册的技能都被 active 白名单过滤掉了
- **AI 意图识别完全失效**：Qwen 收到空 tools 列表，只能返回文字回复
- **前端用户体验**：看到通用回复或超时后显示"出了点问题"

### 11.6 修复方案

**核心原则：admin DB 是 single source of truth，硬编码注册应迁移为动态加载。**

| 编号 | 措施 | 状态 |
|------|------|------|
| F1 | 移除 `bootstrap.ts` 中 `view_land_plot` 的硬编码注册（`RegistryBackedLandPlotProvider` + `createViewLandPlotSkill`），由 `SkillLoader` 通过 `plot_list` 的 `implementation_path` 动态加载 | ✅ 已完成 |
| F2 | 修复 DB `admin_test.skills` 中 `plot_list` 的 `platform_source` 从 `Aigis` 改为 `General` | ✅ 已完成 |
| F3 | 修复 `plot_list.ts:99` 的 `"invalid_input"` 改为 `"unknown"`（`SkillErrorCategory` 不包含 `invalid_input`） | ✅ 已完成 |
| F4 | 清理重复的 `deleted` 记录（`DELETE FROM admin_test.skills WHERE status='deleted'`） | 待执行 |
| F5 | 在 `getFilteredSkills()` 中添加日志：当过滤后为空时，输出 WARN 含已注册 skill 数量和 activeSkillNames | 待实施 |
| F6 | 在 `AIIntentHandler.handle()` 中，当 `tools.length === 0` 时输出 WARN 日志 | 待实施 |
| F7 | 增加启动 smoke test：验证 `toFilteredToolDefinitions()` 返回非空数组 | 待实施 |
