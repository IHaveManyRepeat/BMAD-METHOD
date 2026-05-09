---
type: openapi-experience
title: "OpenAPI 校验：nullable 必须搭配 type"
date: 2026-05-09
tags: [openapi, validation, redocly, nullable, schema]
tech_stack:
  backend: Hono
  language: TypeScript
category: pitfall
applicability: [frontend-backend-separated]
source_project: "diy-a2ui"
---

# OpenAPI 校验：nullable 必须搭配 type

## 问题

使用 `@redocly/cli lint` 校验时出现 `nullable-type-sibling` 错误（7 处）。

## 错误示例

```yaml
# 错误：有 nullable 没有 type
default_value:
  description: "默认值"
  nullable: true
```

## 正确写法

```yaml
default_value:
  type: string        # 必须显式声明 type
  description: "默认值"
  nullable: true
```

## 原因

OpenAPI 3.0.x 规范中 `nullable` 不是 `type` 的替代品，它只是表示该字段可以为 `null`。必须同时声明 `type` 来定义非 null 时的类型。
