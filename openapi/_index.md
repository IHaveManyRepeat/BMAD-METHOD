# OpenAPI Experience Index

Auto-generated index of OpenAPI experience entries.

## Frontmatter Schema

```yaml
---
type: openapi-experience
title: "Title"
date: YYYY-MM-DD
tags: [openapi, ...]
tech_stack:
  backend: framework-name
  language: language-name
category: pitfall | best-practice | pattern | tool-tip
applicability: [frontend-backend-separated]
source_project: "Project name (optional)"
---
```

## Entries

<!-- Entries are auto-generated. Do not edit manually. -->

| Date | Title | Category | Tags |
|------|-------|----------|------|
| 2026-05-09 | [POST 创建接口的响应应保持轻量](2026-05-09-post-response-keep-lightweight.md) | best-practice | openapi, rest-api, post-response |
| 2026-05-09 | [当字段编辑权限受状态约束时，提供两种粒度的编辑接口](2026-05-09-dual-granularity-edit-endpoints.md) | pattern | openapi, state-machine, put-endpoint |
| 2026-05-09 | [参数明细文档必须内联展开所有 $ref 引用](2026-05-09-inline-ref-in-api-docs.md) | best-practice | openapi, documentation, $ref |
| 2026-05-09 | [OpenAPI 校验：nullable 必须搭配 type](2026-05-09-nullable-requires-type-sibling.md) | pitfall | openapi, validation, nullable |
| 2026-05-09 | [接口编号在拆分/新增后必须同步更新](2026-05-09-sync-endpoint-numbering.md) | pitfall | openapi, documentation, consistency |

## Categories

- **pitfall** — Common mistakes and how to avoid them
- **best-practice** — Proven approaches for OpenAPI generation
- **pattern** — Reusable API design patterns
- **tool-tip** — Tool-specific tips and configurations
