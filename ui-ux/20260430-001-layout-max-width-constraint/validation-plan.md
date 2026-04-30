# Validation Plan

## Validation Plan: 管理端内容区布局宽度限制错误

---

## 1. 静态分析

| 工具 | 规则 | 描述 |
|------|------|------|
| 自定义脚本 | 检测布局组件中的 max-width/mx-auto 组合 | 扫描 `layouts/` 目录，发现 `max-w-[xxx] mx-auto` 模式时报错 |
| CSS 审计 | 对比设计规范 token 与实际使用 | 确保布局组件不引入设计规范未定义的宽度限制 |

### 验证方法

1. 编写脚本扫描 `admin-frontend/src/shared/layouts/` 目录下所有文件
2. 正则匹配 `max-w-\[` 或 `max-width:` + `mx-auto` 的组合
3. 发现匹配项时与 `ux-design-specification.md` 中的布局原则交叉验证

---

## 2. 单元测试

### 测试文件（需新建）

| 测试文件 | 测试内容 |
|----------|----------|
| `admin-frontend/src/shared/layouts/__tests__/AppLayout.test.tsx` | 验证 AppLayout 内容容器无 max-width 限制 |

### 测试场景

1. **TC-001: 内容容器无 max-width 内联样式**
   - 渲染 AppLayout
   - 获取内容区 `<div>`（`<Outlet />` 的父容器）
   - 断言 `className` 不包含 `max-w-[`

2. **TC-002: 内容容器宽度为弹性填满**
   - 渲染 AppLayout
   - 获取内容区 `<div>`
   - 断言包含 `flex-1` 和 `w-full`（或等价样式）

3. **TC-003: 内容容器无居中偏移**
   - 渲染 AppLayout
   - 获取内容区 `<div>`
   - 断言 `className` 不包含 `mx-auto`

### 测试代码示例

```tsx
// admin-frontend/src/shared/layouts/__tests__/AppLayout.test.tsx
import { describe, it, expect, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import { createElement } from 'react'
import { MemoryRouter, Routes, Route } from 'react-router-dom'
import { AppLayout } from '../AppLayout'

// Mock auth store
vi.mock('@/features/auth/hooks/useAuthStore', () => ({
  useAuthStore: () => ({
    logout: vi.fn(),
    user: { username: 'test-admin' },
  }),
}))

// Mock GlobalSearch
vi.mock('@/shared/components/GlobalSearch', () => ({
  GlobalSearch: () => null,
}))

function renderWithRouter() {
  return render(
    createElement(MemoryRouter, { initialEntries: ['/'] },
      createElement(Routes, null,
        createElement(Route, {
          path: '/',
          element: createElement(AppLayout, null,
            createElement('div', { 'data-testid': 'page-content' }, '测试页面'),
          ),
        }),
      ),
    ),
  )
}

describe('AppLayout', () => {
  it('内容容器不应有 max-width 限制', () => {
    renderWithRouter()
    const contentArea = screen.getByTestId('page-content').parentElement!
    expect(contentArea.className).not.toContain('max-w-[')
  })

  it('内容容器不应有 mx-auto 居中', () => {
    renderWithRouter()
    const contentArea = screen.getByTestId('page-content').parentElement!
    expect(contentArea.className).not.toContain('mx-auto')
  })

  it('内容容器应弹性填满宽度', () => {
    renderWithRouter()
    const contentArea = screen.getByTestId('page-content').parentElement!
    expect(contentArea.className).toContain('flex-1')
    expect(contentArea.className).toContain('w-full')
  })
})
```

---

## 3. E2E 测试

### 测试场景

| 测试文件 | 测试内容 |
|----------|----------|
| `admin-frontend/e2e/layout-compliance.spec.ts`（需新建） | 验证布局宽度符合设计规范 |

### 测试场景

1. **宽屏下内容区弹性填满**
   - 设置视口宽度为 1920px
   - 登录后访问首页
   - 获取内容区域的 `boundingBox`
   - 断言内容区宽度 ≈ 视口宽度 - 侧边栏宽度（240px）

2. **1280px 最小宽度下内容区弹性填满**
   - 设置视口宽度为 1280px
   - 验证内容区宽度 ≈ 1040px（1280 - 240）

### E2E 测试代码示例

```ts
// admin-frontend/e2e/layout-compliance.spec.ts
import { test, expect } from '@playwright/test'

test.describe('布局宽度合规性', () => {
  test.use({ viewport: { width: 1920, height: 1080 } })

  test('1920px 视口下内容区弹性填满（无 max-width 限制）', async ({ page }) => {
    // 登录后访问首页
    await page.goto('/')
    // ... 登录步骤 ...

    // 获取内容区域
    const contentArea = page.locator('.flex-1.overflow-auto').first()
    const box = await contentArea.boundingBox()

    // 1920 - 240(侧边栏) = 1680，允许 ±10px 容差
    expect(box!.width).toBeGreaterThan(1660)
    expect(box!.width).toBeLessThan(1700)
  })
})
```

---

## 4. 验证清单

- [ ] `AppLayout.tsx` 中移除 `max-w-[1200px] mx-auto`
- [ ] 设计原型 `02-layout-dashboard.html` 移除 `max-width: 1200px`
- [ ] 单元测试 `AppLayout.test.tsx` 创建并通过
- [ ] E2E 测试 `layout-compliance.spec.ts` 创建并通过
- [ ] 视觉验证：1920px 屏幕截图确认内容区填满
- [ ] 视觉验证：1280px 屏幕截图确认内容区填满
