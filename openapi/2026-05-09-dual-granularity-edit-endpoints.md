---
type: openapi-experience
title: "当字段编辑权限受状态约束时，提供两种粒度的编辑接口"
date: 2026-05-09
tags: [openapi, rest-api, api-design, state-machine, put-endpoint]
tech_stack:
  backend: Hono
  language: TypeScript
category: pattern
applicability: [frontend-backend-separated]
source_project: "diy-a2ui"
---

# 当字段编辑权限受状态约束时，提供两种粒度的编辑接口

## 适用条件（必须同时满足）

1. **资源有状态机**，且不同状态下的可编辑字段范围不同
2. **存在"任何状态都可编辑"的基础字段**（如名称、描述）——修改这些字段不影响运行状态
3. **存在"仅特定状态可编辑"的运行时字段**（如依赖接口、参数映射、平台来源）——修改这些字段可能影响系统运行

只有满足上述条件时，才需要拆分。如果所有字段在任何状态下都可编辑，一个 `PUT` 接口就够了。

## 问题

初始设计只有一个 `PUT /skills/{id}` 接口，只包含 `name` 和 `description`。用户反馈"还缺少编辑元信息以外的其他信息接口"。

本项目的 Skill 有状态机（registered → active → disabled → ...），`active` 状态下运行时字段被锁定不可修改，但名称/描述在任何状态都可以改。

## 纠正

拆分为两个接口：

| 接口 | 路径 | 可编辑字段 | 状态限制 | 用途 |
|------|------|-----------|---------|------|
| 轻量编辑 | `PUT /skills/{id}` | name, description | **任何状态** | 改名称/描述，不影响运行 |
| 完整配置 | `PUT /skills/{id}/config` | name, description, platform, dependency_api_ids, params | **仅 registered/disabled/validation_failed** | 全量配置，影响运行 |

## 设计要点

- 轻量接口 = 任何状态都可调用的字段集合（不含运行时字段）
- 完整配置接口 = 仅非运行状态下可调用的字段集合（包含运行时字段）
- 两个接口的**状态限制不同**是拆分的核心原因，而非"前端有两种页面"
- 增量操作仍用专用子接口（如 `POST /resources/{id}/dependencies` 增删单个依赖）
- 文档中明确标注每个接口的状态限制和"全量覆盖"语义
