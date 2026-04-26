# Workflow Change Proposal — Design Fidelity Gate 硬化

## 变更概述

将 BMad Pipeline 中所有"软门禁"（markdown 文档）升级为"硬门禁"（可执行脚本 + 必填审计字段）。

## 变更内容

### 1. Design Fidelity Gate 改造

**当前**：
- `design-fidelity-gate.md` 是纯文档
- Agent 读取后自声称通过
- 无审计字段

**改为**：
- 增加"执行 Playwright 视觉回归测试"的步骤
- 使用 `toHaveScreenshot()` 做像素级对比
- 输出结构化 JSON 报告到指定路径
- 退出码非零时 BLOCK

### 2. Pipeline State 模板硬化

**当前** `step-03-story-loop.md` 中的 story_results 模板：
```yaml
- story_key: "..."
  status: "..."
  coverage: "..."
  review: "..."
```

**改为**（增加必填字段）：
```yaml
- story_key: "..."
  status: "..."
  coverage: "..."
  review: "..."
  fidelity_result: "pass | fail | skipped"   # 新增必填
  fidelity_match_rate: "100.0%"               # 新增必填
  fidelity_differences: 0                     # 新增必填
```

### 3. 门禁执行模式分类

| 门禁类型 | 当前模式 | 建议模式 |
|----------|----------|----------|
| Coverage | 可执行（vitest） | 保持 |
| Review | Agent 自声称 | 保持（需人工判断） |
| Design Fidelity | Agent 自声称 | **Playwright 自动化** |
| Doc Sync | Agent 自声称 | 保持 |
| API Contract | 部分可执行 | 逐步自动化 |

## 实施优先级

1. **P0**：Playwright 视觉回归测试基础设施
2. **P1**：登录页 fidelity 测试（作为试点）
3. **P2**：pipeline-state.yaml 硬化
4. **P3**：其他页面的 fidelity 测试
