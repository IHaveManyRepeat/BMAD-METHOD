# UI/UX Bug Registry

此目录记录 UI/UX 相关 bug 的分析报告。

---

## Bug Reports

| ID | Date | Title | Status | Root Cause |
|----|------|-------|--------|------------|
| 20260506-004 | 2026-05-06 | 接口详情页的参数定义缺乏编辑功能 | 功能实现不完整 — ApiForm 缺少 params_schema 编辑 |

---

## Entries

### 20260506-004 — 接口详情页的参数定义缺乏编辑功能

**类型**: ui-ux (major)
**分析日期**: 2026-05-06

**摘要**:
接口详情页面中参数定义（params_schema）只读显示，编辑表单缺少该字段的编辑功能。

**根本原因**:
ApiForm 组件在实现时遗漏了 params_schema 和 response_schema 字段的编辑支持。

**修复方案**:
- 使用 `story` 路径，创建新的功能改进 story
- 在 ApiForm 中添加 JSON 编辑器用于参数定义
- 添加完整的单元测试和集成测试

**相关文档**:
- [report.md](./20260506-004-params-edit-missing/report.md)
- [validation-plan.md](./20260506-004-params-edit-missing/validation-plan.md)

---

*Last updated: 2026-05-06*
