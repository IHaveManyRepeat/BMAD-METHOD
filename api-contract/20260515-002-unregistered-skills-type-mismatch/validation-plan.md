# 验证计划

## 概述

此验证计划旨在防止 UnregisteredSkillBanner 类型不匹配 bug 再次发生。

## 1. 单元测试

### 1.1 后端单元测试

**文件**: `admin-backend/src/features/skill/service.rs`

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_get_unregistered_skills_returns_correct_structure() {
        // 测试返回结构是否包含 skills 和 total 字段
    }

    #[test]
    fn test_get_unregistered_skills_empty_when_no_skills() {
        // 测试没有未注册技能时返回空数组
    }
}
```

### 1.2 前端单元测试

**文件**: `admin-frontend/src/features/skills/hooks/useSkills.test.ts`

```typescript
describe('useUnregisteredSkills', () => {
  it('should handle response with UnregisteredSkillsResponse structure', () => {
    // 测试正确解析 { skills: [], total: 0 } 结构
  })

  it('should handle empty skills array', () => {
    // 测试处理空数组情况
  })
})
```

## 2. 集成测试

### 2.1 API 契约测试

**文件**: `admin-backend/tests/integration_tests.rs`

```rust
#[tokio::test]
async fn test_unregistered_skills_api_response_format() {
    // 测试 GET /api/admin/skills/unregistered 返回正确的 JSON 结构
    let response = client.get("/api/admin/skills/unregistered")
        .send()
        .await;

    let json: serde_json::Value = response.json().await;
    assert!(json["skills"].is_array());
    assert!(json["total"].is_number());
}
```

### 2.2 前端集成测试

**文件**: `admin-frontend/src/features/skills/components/UnregisteredSkillBanner.test.tsx`

```typescript
describe('UnregisteredSkillBanner', () => {
  it('should display count correctly', () => {
    // 测试显示正确的技能数量
  })

  it('should not render when no unregistered skills', () => {
    // 测试没有未注册技能时不渲染
  })

  it('should render skill list when expanded', () => {
    // 测试展开时显示技能列表
  })
})
```

## 3. E2E 测试

**文件**: `admin-frontend/tests/e2e/unregistered-skills.spec.ts`

```typescript
import { test, expect } from '@playwright/test'

test.describe('Unregistered Skills', () => {
  test('should display banner with correct count', async ({ page }) => {
    await page.goto('/skills')
    // 验证横幅显示正确的数量
  })

  test('should expand and show skills list', async ({ page }) => {
    await page.goto('/skills')
    // 点击横幅展开
    // 验证显示技能列表
  })

  test('should hide banner when no unregistered skills', async ({ page }) => {
    // 先导入所有技能
    // 刷新页面
    // 验证横幅不显示
  })
})
```

## 4. 静态分析

### 4.1 TypeScript 类型检查

- 启用 `tsc --noEmit` 确保类型安全
- 确保 `UnregisteredSkill` 接口与后端模型一致

### 4.2 Rust 类型检查

- 使用 `cargo clippy -- -D warnings` 进行 lint 检查
- 确保响应结构正确实现 `Serialize` 和 `Deserialize`

### 4.3 接口契约一致性检查

**新建工具**: `tools/check-api-contract.js`

```javascript
// 自动检查前后端接口类型一致性
// 从 OpenAPI 规范生成前端类型
```

## 5. 运行时检查

### 5.1 前端数据验证

使用 `zod` 进行运行时数据验证：

```typescript
import { z } from 'zod'

const UnregisteredSkillsResponseSchema = z.object({
  skills: z.array(z.object({
    skill_id: z.string(),
    description: z.string(),
    file_path: z.string(),
  })),
  total: z.number(),
})

// 在 API 响应处理中使用
const validated = UnregisteredSkillsResponseSchema.parse(response.data)
```

### 5.2 后端响应验证

确保响应结构始终符合契约：

```rust
pub fn get_unregistered_skills(
    pool: &PgPool,
    backend_skill_dir: &Path,
) -> Result<UnregisteredSkillsResponse, AppError> {
    // ... 实现逻辑

    // 确保返回结构正确
    Ok(UnregisteredSkillsResponse {
        skills: unregistered,
        total: unregistered.len(),
    })
}
```

## 6. 预防措施

1. **OpenAPI 规范**：为所有 API 端点生成 OpenAPI 规范文档
2. **自动类型生成**：从 OpenAPI 规范自动生成前端 TypeScript 类型
3. **契约测试**：添加 Pact 或类似工具进行契约测试
4. **CI 检查**：在 CI 中添加接口一致性检查