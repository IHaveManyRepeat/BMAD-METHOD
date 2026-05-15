# 工作流变更提案

## 当前问题

在开发过程中，前端 API 类型定义与后端实际响应结构不匹配，导致运行时错误。具体表现为：

1. 后端正确返回 `UnregisteredSkillsResponse { skills: [...], total: 2 }`
2. 前端 API 使用错误的泛型 `<UnregisteredSkill[]>`，导致 TypeScript 类型检查通过但运行时数据解析错误
3. 组件接收到响应对象而不是数组，导致 `.map()` 崩溃
4. 组件直接使用 `skills.length` 而不是 `total`，显示不正确的数量

## 提议的变更

### 1. 修复前端 API 泛型使用规范

**当前问题**：
```typescript
// ❌ 错误：泛型误导 TypeScript，但不影响实际数据
apiClient.get<UnregisteredSkill[]>('/skills/unregistered')
// response.data 实际是 { skills: [...], total: ... }
```

**改进后**：
```typescript
// ✅ 正确：不使用泛型，让 axios 根据响应推断
apiClient.get('/skills/unregistered')
// response.data 将是实际的响应对象
```

**实施规则**：
1. **禁止在 `apiClient.get()` 中使用泛型**，除非后端确实返回简单类型（如纯数组）
2. **为复杂的响应定义接口**，明确描述响应结构
3. **使用响应包装接口**，而不是直接假设类型

### 2. 建立 API 响应类型定义标准

**当前规范**：缺乏统一的 API 响应类型定义标准

**改进后**：
```typescript
// 为每个 API 端点定义明确的响应类型
export interface UnregisteredSkillsResponse {
  skills: UnregisteredSkill[]
  total: number
}

// 在 API 方法中使用正确的类型
export const skillsApi = {
  getUnregistered: async (): Promise<UnregisteredSkillsResponse> => {
    const response = await apiClient.get('/skills/unregistered')
    return response.data
  },
}
```

### 3. 添加运行时数据验证机制

**当前流程**：完全依赖 TypeScript 编译时类型，没有运行时验证

**改进后**：
```typescript
import { z } from 'zod'

// 定义验证 schema
const UnregisteredSkillsResponseSchema = z.object({
  skills: z.array(z.object({
    skill_id: z.string(),
    description: z.string(),
    file_path: z.string(),
  })),
  total: z.number(),
})

// 在 API 客户端中使用
export const skillsApi = {
  getUnregistered: async (): Promise<UnregisteredSkillsResponse> => {
    const response = await apiClient.get('/skills/unregistered')

    // ✅ 运行时验证
    const validated = UnregisteredSkillsResponseSchema.parse(response.data)
    return validated
  },
}
```

### 4. 组件数据访问安全规范

**当前规范**：组件直接访问数据，没有验证数据结构

**改进后**：
```typescript
const { data: unregisteredSkills } = useUnregisteredSkills()

// ✅ 添加类型守卫和可选链访问
const skills = unregisteredSkills?.skills ?? []
const total = unregisteredSkills?.total ?? 0

// ✅ 添加数组类型检查
const skillsArray = Array.isArray(skills) ? skills : []
const safeCount = skillsArray.length

// ✅ 使用 total 字段而不是 skills.length
const displayCount = total
```

### 5. 建立 API 文档和类型同步机制

**当前流程**：前后端类型定义分散且未同步

**改进后**：
1. **后端生成 OpenAPI 规范**：
   - 使用 `utoipa` 或 `utoip-swagger` 自动生成 OpenAPI 文档
   - 将 OpenAPI 文档提交到 `docs/openapi/` 目录

2. **前端从 OpenAPI 生成类型**：
   - 使用 `openapi-typescript` 或类似工具自动生成类型
   - 将生成的类型提交到 `admin-frontend/src/types/generated/`

3. **CI 检查**：
   - 在 PR 审查时检查前后端类型是否一致
   - 添加契约测试确保 API 响应结构符合规范

### 6. 添加开发指南和最佳实践

**当前问题**：缺乏明确的 API 开发指南

**改进后**：建立 API 开发最佳实践文档

**禁止模式**：
```typescript
// ❌ 禁止：在 apiClient.get() 中使用泛型
apiClient.get<SomeType[]>('/api/endpoint')

// ❌ 禁止：假设响应类型而不定义接口
const response = await apiClient.get('/api/endpoint')
const skills = response as any[]  // 危险的类型断言

// ❌ 禁止：直接访问响应属性而不验证
const count = response.skills?.length  // 可能未定义
```

**推荐模式**：
```typescript
// ✅ 推荐：定义明确的响应接口
export interface ApiResponse {
  data: DataType
  meta: ResponseMeta
}

// ✅ 推荐：让 axios 推断类型
const response = await apiClient.get('/api/endpoint')
const validated = Schema.parse(response.data)

// ✅ 推荐：添加类型守卫
const data = response.data ?? defaultValue
const array = Array.isArray(data) ? data : []
```

## 影响

### 正向影响

1. **Bug 预防**：运行时数据验证可以提前捕获类型不匹配
2. **代码质量提升**：明确的 API 响应类型定义提高代码可维护性
3. **团队协作改进**：OpenAPI 规范确保前后端使用统一的接口定义
4. **开发体验提升**：自动类型生成减少手动同步工作

### 潜在风险

1. **初期学习成本**：团队需要学习新的 API 开发规范
2. **工具链复杂度增加**：引入 zod 和 OpenAPI 生成工具
3. **性能影响**：运行时数据验证可能略微影响性能（可忽略）

## 迁移路径

### 阶段 1：修复当前 bug（立即）

1. 修正 `skillsApi.getUnregistered()` 的类型定义
2. 更新 `UnregisteredSkillBanner` 组件的数据访问逻辑
3. 添加基本的错误边界处理

### 阶段 2：建立规范（1 周）

1. 制定 API 响应类型定义标准文档
2. 编写 API 开发最佳实践指南
3. 在团队内进行培训和分享

### 阶段 3：实施验证机制（1-2 周）

1. 引入 zod 运行时验证
2. 为关键 API 端点添加数据验证
3. 建立 API 类型一致性检查脚本

### 阶段 4：引入自动化工具（2-4 周）

1. 配置后端 OpenAPI 生成工具
2. 设置前端自动类型生成流程
3. 在 CI 中添加契约检查步骤

### 阶段 5：持续改进（长期）

1. 定期审查规范的有效性
2. 根据团队反馈进行调整
3. 探索更高级的工具和机制
4. 建立性能和质量指标