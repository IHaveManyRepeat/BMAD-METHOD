# 验证计划

## 概述

此验证计划旨在防止 UnregisteredSkillBanner API 响应类型解析错误再次发生。

## 1. 单元测试

### 1.1 前端 API 客户端测试

**文件**: `admin-frontend/src/features/skills/api/skillsApi.test.ts`

```typescript
describe('skillsApi.getUnregistered', () => {
  it('should return UnregisteredSkillsResponse structure', async () => {
    const mockAxios = {
      get: jest.fn().mockResolvedValue({
        data: {
          skills: [
            {
              skill_id: 'test-skill',
              description: 'Test description',
              file_path: '/path/to/file.ts',
            },
          ],
          total: 1,
        },
      }),
    }

    const result = await skillsApi.getUnregistered()

    expect(result).toEqual({
      skills: expect.any(Array),
      total: expect.any(Number),
    })
  })

  it('should handle empty skills array', async () => {
    const mockResponse = {
      skills: [],
      total: 0,
    }

    // 测试空数组处理
    const result = await skillsApi.getUnregistered()

    expect(result.skills).toEqual([])
    expect(result.total).toBe(0)
  })
})
```

### 1.2 前端 Hook 测试

**文件**: `admin-frontend/src/features/skills/hooks/useSkills.test.ts`

```typescript
describe('useUnregisteredSkills', () => {
  it('should return UnregisteredSkillsResponse from query', () => {
    // 测试 hook 正确返回响应对象
    const { result } = renderHook(() => useUnregisteredSkills(), {
      wrapper: QueryClientProvider,
    })

    expect(result.current).toMatchObject({
      data: expect.objectContaining({
        skills: expect.any(Array),
        total: expect.any(Number),
      }),
      isLoading: expect.any(Boolean),
    })
  })
})
```

## 2. 集成测试

### 2.1 API 响应结构测试

**文件**: `admin-frontend/src/features/skills/api/skillsApi.test.ts`

```typescript
describe('skillsApi - API Response Validation', () => {
  it('should correctly parse UnregisteredSkillsResponse from backend', async () => {
    // 模拟后端实际返回的响应结构
    const backendResponse = {
      skills: [
        {
          skill_id: 'skill-1',
          description: 'Description 1',
          file_path: '/path/skill-1.ts',
        },
      ],
      total: 1,
    }

    // 验证前端正确解析响应
    const result = await skillsApi.getUnregistered()

    expect(result).toEqual(backendResponse)
    expect(result.skills).toHaveLength(1)
    expect(result.total).toBe(1)
  })
})
```

### 2.2 组件集成测试

**文件**: `admin-frontend/src/features/skills/components/UnregisteredSkillBanner.test.tsx`

```typescript
describe('UnregisteredSkillBanner - Integration', () => {
  it('should display correct count from UnregisteredSkillsResponse', () => {
    const mockResponse = {
      skills: [
        { skill_id: 'skill-1', description: 'Desc', file_path: '/path' },
        { skill_id: 'skill-2', description: 'Desc', file_path: '/path' },
      ],
      total: 2,
    }

    render(
      <QueryClientProvider client={queryClient}>
        <UnregisteredSkillBanner />
      </QueryClientProvider>,
    )

    // 验证显示 total 字段，不是对象属性个数
    expect(screen.getByText(/发现 2 个未注册 Skill/)).toBeInTheDocument()
  })

  it('should not render when total is 0', () => {
    const mockResponse = {
      skills: [],
      total: 0,
    }

    render(
      <QueryClientProvider client={queryClient}>
        <UnregisteredSkillBanner />
      </QueryClientProvider>,
    )

    // 验证没有未注册技能时不显示横幅
    expect(screen.queryByText(/发现.*个未注册 Skill/)).not.toBeInTheDocument()
  })
})
```

## 3. E2E 测试

**文件**: `admin-frontend/tests/e2e/unregistered-skills.spec.ts`

```typescript
import { test, expect } from '@playwright/test'

test.describe('Unregistered Skills', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/skills')
  })

  test('should display banner with correct count from total field', async ({ page }) => {
    // 验证横幅显示后端返回的 total 字段
    const banner = page.locator('[data-testid="unregistered-skills-banner"]')
    await expect(banner).toBeVisible()

    const countText = await banner.locator('.count-text').textContent()
    expect(countText).toMatch(/发现 \d+ 个未注册 Skill/)
  })

  test('should expand and show skills list when clicked', async ({ page }) => {
    // 点击横幅展开
    const banner = page.locator('[data-testid="unregistered-skills-banner"]')
    await banner.click()

    // 验证显示技能列表
    const skillList = page.locator('[data-testid="unregistered-skills-list"]')
    await expect(skillList).toBeVisible()

    // 验证每个技能都正确显示
    const skillItems = page.locator('[data-testid^="unregistered-skill-"]')
    const count = await skillItems.count()
    expect(count).toBeGreaterThan(0)
  })

  test('should hide banner when no unregistered skills (total=0)', async ({ page }) => {
    // 先导入所有技能（设置 total=0）
    // 刷新页面
    await page.reload()

    // 验证横幅不显示
    const banner = page.locator('[data-testid="unregistered-skills-banner"]')
    await expect(banner).not.toBeVisible()
  })

  test('should correctly handle batch import', async ({ page }) => {
    const banner = page.locator('[data-testid="unregistered-skills-banner"]')
    await banner.click()

    // 全选技能
    const selectAll = page.locator('[data-testid="select-all-checkbox"]')
    await selectAll.check()

    // 点击批量导入
    const importButton = page.locator('[data-testid="batch-import-button"]')
    await importButton.click()

    // 验证导入成功后横幅消失
    await expect(banner).not.toBeVisible()

    // 验证页面没有崩溃
    await page.waitForLoadState('networkidle')
    expect(page.locator('body')).not.toHaveClass(/crash|error/)
  })
})
```

## 4. 静态分析

### 4.1 TypeScript 类型检查

- 启用 `tsc --noEmit` 确保类型安全
- 确保 `UnregisteredSkillsResponse` 接口定义正确

### 4.2 Lint 规则

```json
// .eslintrc.json
{
  "rules": {
    "@typescript-eslint/no-unsafe-member-access": "error",
    "@typescript-eslint/strict-boolean-expressions": "error"
  }
}
```

### 4.3 API 类型一致性检查

**新建工具**: `tools/check-api-contract.js`

```javascript
// 自动检查前端 API 类型定义是否与后端响应结构匹配
// 通过解析后端 Rust 结构和前端 TypeScript 接口进行对比
// 报告类型不匹配的字段
```

## 5. 运行时检查

### 5.1 前端数据验证

使用 `zod` 进行运行时数据验证：

```typescript
import { z } from 'zod'

const UnregisteredSkillSchema = z.object({
  skill_id: z.string(),
  description: z.string(),
  file_path: z.string(),
})

const UnregisteredSkillsResponseSchema = z.object({
  skills: z.array(UnregisteredSkillSchema),
  total: z.number(),
})

// 在 API 响应处理中使用
export const skillsApi = {
  getUnregistered: async (): Promise<UnregisteredSkillsResponse> => {
    const response = await apiClient.get('/skills/unregistered')

    // ✅ 添加运行时验证
    const validated = UnregisteredSkillsResponseSchema.parse(response.data)
    return validated
  },
}
```

### 5.2 组件数据访问验证

在组件中添加类型守卫：

```typescript
const { data: unregisteredSkills, isLoading } = useUnregisteredSkills()

// ✅ 确保 unregisteredSkills 有正确的结构
const skills = unregisteredSkills?.skills ?? []
const total = unregisteredSkills?.total ?? 0

// ✅ 添加边界检查
if (!Array.isArray(skills)) {
  console.error('Expected skills to be array, got:', typeof skills)
  return <div>数据格式错误</div>
}

const safeCount = Number.isInteger(total) ? total : 0
```

## 6. 预防措施

1. **OpenAPI 规范**：为所有 API 端点生成 OpenAPI 规范文档
2. **自动类型生成**：从 OpenAPI 规范自动生成前端 TypeScript 类型
3. **契约测试**：添加 API 响应结构测试
4. **CI 检查**：在 CI 中添加类型一致性检查
5. **运行时验证**：使用 zod 或类似库进行数据验证
6. **组件防御性编程**：添加类型守卫和错误边界
