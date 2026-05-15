# 工作流变更提案

## 当前问题

在开发过程中，前后端接口类型不一致的问题没有被及时发现。具体表现为：

1. 后端定义了 `UnregisteredSkillsResponse { skills, total }` 结构
2. 前端期望直接返回 `UnregisteredSkill[]` 数组
3. 字段名不匹配：`skill_id` vs `id`, `file_path` vs `source_path`
4. 缺少运行时数据验证，导致页面崩溃

## 提议的变更

### 1. 引入 OpenAPI 规范流程

**当前流程**：
```
后端开发 → 前端开发 → 手动类型同步 → 测试
```

**改进后流程**：
```
后端开发 → 生成 OpenAPI 规范 → 前端从规范生成类型 → 自动验证 → 测试
```

**具体实施**：
1. 后端接口开发时使用 `utoipa` 或 `utoip-swagger` 自动生成 OpenAPI 文档
2. 将 OpenAPI 文档提交到 `docs/openapi/` 目录
3. 前端使用 `openapi-typescript` 或类似工具自动生成类型定义
4. 添加 CI 检查步骤来验证类型一致性

### 2. 添加接口契约验证步骤

在开发工作流中添加以下检查点：

**Story 创建阶段**：
- [ ] 后端开发完成后更新 OpenAPI 文档
- [ ] 前端从更新的 OpenAPI 生成类型

**Story 验证阶段**：
- [ ] 运行接口契约一致性检查脚本
- [ ] 手动验证前后端类型定义是否匹配
- [ ] 测试正常和边界情况（空数组、错误响应等）

**PR 审查阶段**：
- [ ] 检查是否更新了 OpenAPI 文档
- [ ] 检查前端类型是否自动生成
- [ ] 运行契约测试确保一致性

### 3. 添加运行时数据验证

**前端 API 客户端改造**：

```typescript
// 使用 zod 或类似库进行运行时验证
import { z } from 'zod'

const UnregisteredSkillsResponseSchema = z.object({
  skills: z.array(z.object({
    id: z.string(),
    name: z.string(),
    description: z.string().nullable(),
    source_path: z.string(),
  })),
  total: z.number(),
})

// 在响应拦截器中自动验证
apiClient.interceptors.response.use((response) => {
  if (response.config.url?.includes('/skills/unregistered')) {
    const validated = UnregisteredSkillsResponseSchema.parse(response.data)
    return { ...response, data: validated }
  }
  return response
})
```

### 4. 建立 API 文档站点

**目的**：提供实时可访问的 API 文档，便于前后端开发人员查阅

**实施方案**：
1. 使用 Swagger UI 或 Redoc 部署 API 文档
2. 配置自动更新机制（OpenAPI 文档变更时重新部署）
3. 在开发服务器中嵌入文档链接

## 影响

### 正向影响

1. **开发效率提升**：自动类型生成减少手动同步工作
2. **Bug 预防**：契约验证提前发现类型不匹配问题
3. **文档一致性**：OpenAPI 规范确保前后端使用统一的接口定义
4. **团队协作**：API 文档站点便于跨团队查阅

### 潜在风险

1. **初期学习成本**：团队需要学习 OpenAPI 工具和流程
2. **工具链复杂度增加**：引入新的工具和 CI 步骤
3. **开发流程变更**：需要适应新的开发习惯

## 迁移路径

### 阶段 1：基础设施搭建（1-2 周）

1. 选择并安装 OpenAPI 生成工具（`utoipa`）
2. 配置前端类型生成工具（`openapi-typescript`）
3. 搭建 API 文档站点（Swagger UI）
4. 编写类型生成和契约验证脚本

### 阶段 2：试点应用（1 周）

1. 选择一个 API 模块（如 skills）作为试点
2. 应用新的工作流和工具
3. 收集反馈并优化流程
4. 编写最佳实践文档

### 阶段 3：全面推广（2-4 周）

1. 逐步将新流程应用到所有 API 模块
2. 培训团队使用新工具和流程
3. 更新开发规范和 PR 审查清单
4. 建立 CI 检查确保流程执行

### 阶段 4：持续改进（长期）

1. 定期审查工具和流程的有效性
2. 根据团队反馈进行调整
3. 探索更高级的工具（如契约测试）
4. 建立性能和质量指标