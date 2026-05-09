---
type: openapi-experience
title: "接口编号在拆分/新增后必须同步更新"
date: 2026-05-09
tags: [openapi, documentation, api-numbering, consistency]
tech_stack:
  backend: Hono
  language: TypeScript
category: pitfall
applicability: [frontend-backend-separated]
source_project: "diy-a2ui"
---

# 接口编号在拆分/新增后必须同步更新

## 问题

拆分 PUT 接口为两个后（1.4 + 1.5），后续接口编号（1.5→1.6, 1.6→1.7...）没有同步更新，导致文档中引用混乱。

## 教训

每次新增/拆分接口后，**立即更新**：

1. 文档目录中的编号和锚点链接
2. 正文中所有"同 X.X 响应结构"的交叉引用
3. 校验无残留的旧编号引用（用 grep 检查）
