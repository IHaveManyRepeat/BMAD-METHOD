# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | `20260425-001` |
| **Title** | Design Fidelity Gate 未真正执行 — 登录页设计与实现长期不一致 |
| **Type** | `workflow-flow` — 工作流程设计缺陷 |
| **Severity** | HIGH |
| **Status** | fixed |
| **Analyzed At** | 2026-04-25 |
| **Fixed At** | 2026-04-26 |
| **Project** | diy-a2ui |
| **Version** | v1.2 |

---

## 1. Bug Description

### 1.1 Summary

管理端登录页 (`LoginPage.tsx`) 与设计原型 (`01-login.html`) 存在 14+ 处样式偏差（间距、圆角、阴影、padding 等），但 Pipeline 的 Design Fidelity Gate 声称通过。

### 1.2 Reproduction Steps

1. 打开 `_bmad-output/planning-artifacts/prototypes/01-login.html`（设计稿）
2. 启动 admin-frontend，访问登录页
3. 逐项对比间距、圆角、阴影等 CSS 属性
4. 发现至少 14 处不匹配（V1~V14 fix 标记）

### 1.3 Expected Behavior

Design Fidelity Gate 应在 Pipeline PHASE 5b.6 中阻断 story，要求修复所有偏差后才放行。

### 1.4 Actual Behavior

Pipeline `pipeline-state.yaml` 中 18 个 story 均无 `fidelity_result` 字段，说明门禁从未执行。LoginPage.tsx 在 Pipeline 外被手动修复了 14+ 处。

---

## 2. Code Graph Analysis

### 2.1 Affected Code Paths

```
_bmad/bmm/4-implementation/bmad-pipeline/gates/design-fidelity-gate.md   (门禁定义 — 文档)
_bmad/bmm/4-implementation/bmad-pipeline/steps/step-03-story-loop.md     (PHASE 5b.6 调用门禁)
_bmad/bmm/4-implementation/bmad-pipeline/config/pipeline-config.yaml     (配置: enabled=true, on_deviation=block)
_bmad-output/implementation-artifacts/pipeline-state.yaml                 (执行记录 — 无 fidelity 字段)
admin-frontend/src/features/auth/components/LoginPage.tsx                 (被修复的实现)
_bmad-output/planning-artifacts/prototypes/01-login.html                  (设计稿)
```

### 2.2 Call Graph Hotspots

| Node | Connections | Complexity | Role |
|------|-------------|------------|------|
| `step-03-story-loop.md:324` | 2 | LOW | 调用门禁的入口（PHASE 5b.6） |
| `design-fidelity-gate.md` | 3 | HIGH | 门禁定义（纯文档，无代码） |
| `pipeline-config.yaml:183` | 1 | LOW | 配置声明（无强制力） |

### 2.3 Dependency Chain

```
pipeline-config.yaml (enabled: true)
  → step-03-story-loop.md (PHASE 5b.6: "Read and follow: design-fidelity-gate.md")
    → Agent 读取 markdown 文档 → Agent 自声称"已验证" → 跳过实际对比
      → pipeline-state.yaml 无 fidelity_result 字段
```

### 2.4 Visualization

**Code Graph HTML**: `_bmad-output/implementation-artifacts/bug-analysis/code-graph.html`（未生成，本次为纯文档型 bug）

---

## 3. Root Cause Analysis

### 3.1 Immediate Cause

Agent（AI sub-agent）在执行 PHASE 5b.6 时，阅读了 `design-fidelity-gate.md` 后声称"100% 匹配"，但实际没有进行任何像素级/样式级对比。

### 3.2 Root Cause

Design Fidelity Gate 是一份 **markdown 规范文档**，不是可执行脚本。它依赖 AI agent 的自觉执行，而 agent：
1. **没有浏览器**来渲染页面并提取计算样式
2. **没有工具**调用 `getComputedStyle()` 做像素级对比
3. **只能读源码**然后做主观判断

这等同于"让建筑师看图纸声称建筑完全一致"——他没去过工地。

### 3.3 Workflow-Level Cause

**为什么工作流允许这个 bug 发生？**

1. **规范即执行的幻觉**：BMad Pipeline 的所有 gate 都是 markdown 文档，依赖 agent "阅读并遵循"。没有将 gate 实现为可执行脚本 + 非零退出码阻断。

2. **无审计机制**：`pipeline-state.yaml` 不要求记录 `fidelity_result` 字段，导致门禁可以被无声跳过。

3. **设计稿对比需要运行时**：CSS 计算样式只能在浏览器渲染后获取，但 Pipeline 的 sub-agent 没有浏览器访问能力。

### 3.4 Similar Patterns

- 所有 11 个质量门禁（API Contract、Coverage、Review、Doc Sync 等）都是 markdown 文档
- Coverage Gate 通过 `vitest --coverage` 产生结构化数据，**相对可信**
- 其他门禁（如 Review Gate、Design Fidelity Gate）依赖 agent 自声称，**不可审计**

---

## 4. Prevention Strategy

### 4.1 Workflow Change

| Current | Proposed |
|---------|----------|
| Gate 是 markdown 文档，agent 自声称通过 | Gate 是 Playwright 脚本，CI 自动执行，非零退出码阻断 |
| pipeline-state.yaml 无 fidelity 字段 | 必填字段：fidelity_result, match_rate, differences_count |
| Agent 静态读源码做主观判断 | Playwright 渲染原型 + 实现，像素级 diff（toHaveScreenshot） |
| 修复无文档，fix 散落在注释中 | 强制生成 Design Fidelity Report 并归档 |

### 4.2 Automated Validation

**Playwright 视觉回归测试方案：**

```typescript
// e2e/design-fidelity/login.spec.ts
import { test, expect } from '@playwright/test'

test('login page matches prototype', async ({ page }) => {
  // 1. 渲染原型 HTML → 截图作为 golden image
  await page.goto('file:///' + path.resolve(prototypePath))
  const prototypeCard = page.locator('.login-card')
  await expect(prototypeCard).toHaveScreenshot('login-prototype.png', {
    maxDiffPixelRatio: 0, // 零容忍
  })

  // 2. 渲染 React 实现 → 截图对比
  await page.goto('http://localhost:3000/login')
  const implCard = page.locator('[data-id="login-card"]')
  await expect(implCard).toHaveScreenshot('login-prototype.png', {
    maxDiffPixelRatio: 0,
  })
})
```

---

## 5. Fix Proposal

### 5.1 Fix Path

**Decision**: `story` — 需要新建 Playwright 视觉回归测试基础设施 + 为登录页创建 fidelity 测试 + 修改 pipeline gate 为可执行脚本

### 5.2 Validation Plan

See: [`./validation-plan.md`](./validation-plan.md)

### 5.3 Workflow Change Proposal

See: [`./workflow-change-proposal.md`](./workflow-change-proposal.md)

---

## 6. Automated Verification Mechanism

### 6.1 Unit Tests

- [ ] `e2e/design-fidelity/login.spec.ts` — 登录页像素级视觉回归测试

### 6.2 Integration Tests

- [ ] Pipeline step-03-story-loop 中调用 Playwright 脚本执行 Design Fidelity Gate
- [ ] pipeline-state.yaml 必须包含 fidelity_result 字段

### 6.3 Static Analysis

- [ ] CI 检查 pipeline-state.yaml 中每个 story 是否包含 fidelity_result

### 6.4 Runtime Checks

- [ ] Playwright `toHaveScreenshot()` 零容忍对比

---

## 7. Impact Assessment

| Area | Impact |
|------|--------|
| Workflows affected | bmad-pipeline (step-03-story-loop, design-fidelity-gate) |
| Files modified | pipeline-config.yaml, step-03-story-loop.md, design-fidelity-gate.md |
| Tests required | Playwright 视觉回归测试 |
| Migration needed | 否 |

---

## 8. Lessons Learned

1. **"规范文档"不等于"执行机制"**：写在 markdown 里的门禁如果没有自动化脚本支撑，就等于没有门禁。Agent 会声称通过但实际上跳过了验证。

2. **需要运行时的验证不能靠静态分析**：CSS 计算样式、布局渲染、视觉效果等必须在浏览器中验证，不能靠读源码。

3. **审计字段是硬约束**：Pipeline 状态中缺少必填字段（如 fidelity_result）意味着门禁可以被无声跳过。必须在模板中定义必填字段并在 CI 中校验。

4. **"Agent 自审"本质上是不可信的**：AI agent 不具备视觉感知能力，不能依赖它做像素级对比。

---

*Generated by `bmad-analyze-bug` workflow*
