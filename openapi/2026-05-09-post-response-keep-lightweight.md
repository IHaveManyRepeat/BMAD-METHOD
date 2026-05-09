---
type: openapi-experience
title: "POST 创建接口的响应应保持轻量"
date: 2026-05-09
tags: [openapi, rest-api, api-design, post-response]
tech_stack:
  backend: Hono
  language: TypeScript
category: best-practice
applicability: [frontend-backend-separated]
source_project: "diy-a2ui"
---

# POST 创建接口的响应应保持轻量

## 问题

初始设计让 `POST /api/admin/skills`（注册新 Skill）返回完整的 `SkillDetail`（~50 字段，含 dependencies、param_mapping、aggregated_schema 等嵌套结构）。

## 纠正

改为只返回轻量确认信息：

```json
{
  "data": {
    "id": "register_user",
    "status": "registered",
    "created_at": "2026-05-09T10:00:00Z"
  }
}
```

## 原因

1. **创建可能触发异步操作**（参数分析、骨架文件生成），返回完整数据可能不一致
2. **前端注册成功后通常跳转**到列表页或详情页，届时会单独 GET 拉取
3. **减少响应体大小**，提升感知性能
4. REST 规范中 POST 返回完整资源是"可以"但不是"应该"

## 适用条件

- 创建操作涉及后续异步处理
- 前端注册后会重新拉取数据
- 资源详情结构复杂（嵌套深、字段多）

## 反面条件（返回完整资源更合适的场景）

- 创建是同步完成的
- 前端需要立即使用返回的数据（不跳转）
- 资源结构简单
