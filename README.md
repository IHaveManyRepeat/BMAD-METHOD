# Bug Analysis Registry — Bug 分析报告注册中心

集中归档所有经过 `bmad-analyze-bug` workflow 分析的 bug 报告。

## 目录结构

```
bug-analysis-registry/
├── workflow-flow/         # 工作流程设计缺陷
├── missing-validation/    # 缺失验证机制
├── state-management/      # 状态管理问题
├── api-contract/          # 接口契约问题
├── performance/           # 性能问题
├── security/              # 安全问题
├── ui-ux/                 # 界面/体验问题
├── integration/           # 集成问题
└── other/                 # 其他问题
```

## 归档命名规范

```
{bug_type}/{YYYYMMDD}-{version}-{short-title}.md
```

## 归档流程

1. `bmad-analyze-bug` 生成分析报告 → `_bmad-output/implementation-artifacts/bug-analysis/`
2. 报告归档到对应的 `{bug_type}/` 子目录
3. 更新子目录 `README.md` 中的文档索引

## Bug 类型分类

| 类型 | 说明 | 常见示例 |
|------|------|---------|
| workflow-flow | 工作流程设计缺陷 | Pipeline 暂停点、步骤顺序不合理、缺失验证步骤 |
| missing-validation | 缺失验证机制 | 输入/输出/状态一致性/边界条件 |
| state-management | 状态管理问题 | 状态不一致、泄漏、竞争、同步失败 |
| api-contract | 接口契约问题 | 前后端不匹配、缺失错误响应、类型不一致 |
| performance | 性能问题 | N+1 查询、内存泄漏、缓存缺失 |
| security | 安全问题 | 注入、XSS、认证绕过、数据泄露 |
| ui-ux | 界面/体验问题 | 引导缺失、提示不友好、交互混乱 |
| integration | 集成问题 | 外部依赖失败、超时、同步失败 |
| other | 其他问题 | 文档缺失、配置错误、工具链 |

## 统计

| 类型 | 数量 |
|------|------|
| workflow-flow | 5 |
| missing-validation | 5 |
| state-management | 2 |
| api-contract | 13 |
| performance | 1 |
| security | 0 |
| ui-ux | 14 |
| integration | 4 |
| other | 1 |
| **总计** | **45** |
