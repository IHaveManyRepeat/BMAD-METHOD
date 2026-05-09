---
type: openapi-experience
title: "参数明细文档必须内联展开所有 $ref 引用"
date: 2026-05-09
tags: [openapi, documentation, api-review, $ref]
tech_stack:
  backend: Hono
  language: TypeScript
category: best-practice
applicability: [frontend-backend-separated]
source_project: "diy-a2ui"
---

# 参数明细文档必须内联展开所有 $ref 引用

## 问题

OpenAPI YAML 中通过 `$ref` 引用 Schema 是好的工程实践，但生成的审阅文档只展示了接口概要（方法+路径+描述），没有展开 Schema 的实际字段。用户反馈："我没有看到每个接口的请求参数和返回参数分别是什么"。

## 纠正

单独生成 `api-params-detail.md`，将所有 `$ref` 引用内联展开，每个接口的请求参数和响应参数都以**表格 + JSON 示例**的方式完整展示。

## 原因

1. **API 契约审阅阶段的核心目标是确认字段定义**，不是确认 URL 路径
2. `$ref` 对机器友好但对人类不友好，审阅者需要看到每个字段的类型、约束、说明
3. JSON 示例比 Schema 定义更直观，帮助审阅者快速理解数据结构

## 实践建议

- OpenAPI YAML 保持 `$ref` 引用（工程正确性）
- 同时生成一份**展开版参数明细文档**供人工审阅
- 明细文档中每个接口包含：请求参数表格 + 请求示例 JSON + 成功响应 JSON + 错误响应说明
