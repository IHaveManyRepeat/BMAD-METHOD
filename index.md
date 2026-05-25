# Bug Analysis Registry Index

Auto-generated index of all bug analysis documents.

## api-contract

- [20260525-003 环境管理页接口数量未展示数据](./api-contract/20260525-003-environment-api-count-missing/report.md) — 2026-05-25 — **Medium**
- [20260515-002 UnregisteredSkillBanner API响应类型解析错误导致崩溃](./api-contract/20260515-002-unregistered-skills-type-mismatch/report.md) — 2026-05-15 — **High**
- [20260514-003 preferences API 返回 500 错误](./api-contract/20260514-003-preferences-api-500/report.md) — 2026-05-14 — **High**
- [20260514-002 页面没有调用接口去查找管理端设置的系统上下文字段](./api-contract/20260514-002-schema-fetch-fail/report.md) — 2026-05-14 — **High**
- [20260514-001 用户端凭证设置未读取管理端系统上下文配置的字段](./api-contract/20260514-001-user-credential-system-context/report.md) — 2026-05-14 — **High**
- [20260513-002 用户端偏好设置无法读取管理端系统上下文字段定义](./api-contract/20260513-002-system-context-unauthorized/report.md) — 2026-05-13 — **High**
- [20260512-001 schema-viewer-array](./api-contract/20260512-001-schema-viewer-array/report.md) — 2026-05-12
- [20260512-001 20260512-001-api-update-500](./api-contract/20260512-001-20260512-001-api-update-500/report.md) — 2026-05-12
- [20260506-005 Test Proxy 请求 422 — 前后端契约不匹配 + 测试未覆盖](./api-contract/20260506-005-test-proxy-422-contract-mismatch/report.md) — 2026-05-06
- [20260428-v1.1 用户设置 API 验证错误](./api-contract/20260428-v1.1-usersettings-validation-error/report.md) — 2026-04-28
- [20260428-v1.1 api-contract-validation-error](./api-contract/20260428-v1.1-api-contract-validation-error/report.md) — 2026-04-28
- [20260427-v1.2 禁用用户时 422 错误](./api-contract/20260427-v1.2-user-status-422/report.md) — 2026-04-27
- [20260426-001 JWT Base64URL 解析失败导致认证 401](./api-contract/20260426-001-jwt-base64url-parse-failure/report.md) — 2026-04-26

## integration

- [20260525-002 parseParamsSchema 未解析 param_groups 导致 API 调用时分组参数全部丢失](./integration/20260525-002-param-groups-not-extracted/report.md) — 2026-05-25 — **High**
- [20260525-001 pino logMethod hook 未调用 method 导致全部日志静默丢失](./integration/20260525-001-pino-logmethod-hook-silent/report.md) — 2026-05-25 — **High**
- [20260514-002 schema-fetch-fail](./integration/20260514-002-schema-fetch-fail/report.md) — 2026-05-14
- [20260513-001 GET /api/admin/system-context-fields returns 500](./integration/20260513-001-system-context-500/report.md) — 2026-05-13 — **High**

## missing-validation

- [20260525-005 接口注册页表格"最后测试"列硬编码显示横杠](./missing-validation/20260525-005-api-list-last-tested-hardcoded/report.md) — 2026-05-25 — **Medium**
- [20260525-004 接口测试页选择环境后未过滤接口列表](./missing-validation/20260525-004-test-proxy-api-env-filter/report.md) — 2026-05-25 — **Medium**
- [20260515-001 系统上下文字段编辑策略缺失：未根据引用状态区分可编辑范围](./missing-validation/20260515-001-field-edit-reference-guard/report.md) — 2026-05-15 — **High**
- [20260513-001 编辑接口后版本历史不记录](./missing-validation/20260513-001-version-history-missing/report.md) — 2026-05-13 — **Medium**
- [20260506-004 API 详情页编辑/禁用按钮不生效 + 管理端全局按钮功能审计](./missing-validation/20260506-004-api-detail-buttons-not-working/report.md) — 2026-05-06

## other

- [20260429-001 数据库表字段缺少注释](./other/20260429-001-db-comments-missing/report.md) — 2026-04-29

## performance

- [20260523-001 点击齿轮触发 4 次 schema 请求](./performance/20260523-001-schema-quad-request/report.md) — 2026-05-23

## state-management

- [20260520-002 usersettings-not-sent](./state-management/20260520-002-usersettings-not-sent/report.md) — 2026-05-20
- [20260427-001 session-persist-after-tab-close](./state-management/20260427-001-session-persist-after-tab-close/report.md) — 2026-04-27

## ui-ux

- [20260525-007 接口列表页测试弹窗缺少参数分组输入](./ui-ux/20260525-007-test-dialog-param-groups-input-missing/report.md) — 2026-05-25 — **High**
- [20260525-006 骨架文件重新生成缺少二次确认](./ui-ux/20260525-006-skeleton-regenerate-no-confirm/report.md) — 2026-05-25 — **Medium**
- [20260516-001 聚合配置 Tab 缺少参数来源配置和必填状态调整 UI](./ui-ux/20260516-001-aggregate-config-missing-param-source/report.md) — 2026-05-16 — **High**
- [20260513-002 版本历史展示问题：布局、变更内容、hover效果](./ui-ux/20260513-002-版本历史展示问题/report.md) — 2026-05-13 — **Medium**
- [20260513-001 接口版本历史展示方式优化](./ui-ux/20260513-001-version-history-display/report.md) — 2026-05-13 — **Medium**
- [20260513-001 system-context 新增字段弹框字段类型不能选择](./ui-ux/20260513-001-system-context-field-type/report.md) — 2026-05-13 — **Medium** — *closed*
- [20260512-001 接口注册页面表格缺少操作栏](./ui-ux/20260512-001-api-list-missing-actions/report.md) — 2026-05-12 — **Medium-High**
- [20260512-001 ResponseSchemaEditor 数组类型 data[] 被错误展示为 string](./ui-ux/20260512-001-response-schema-array-field-display/report.md) — 2026-05-12 — **High**
- [20260512-001 测试弹框和测试Tab的请求参数应自动填充模板](./ui-ux/20260512-001-测试弹框和测试tab的请求参数应自动填充模板/report.md) — 2026-05-12
- [20260506-004 params-edit-missing](./ui-ux/20260506-004-params-edit-missing/report.md) — 2026-05-06
- [20260506-003 全站按钮颜色硬编码导致颜色不搭配](./ui-ux/20260506-003-button-color-hardcoded-tokens/report.md) — 2026-05-06
- [20260430-003 z-index token 命名空间错误导致 @theme 定义无效](./ui-ux/20260430-003-z-index-token-namespace/report.md) — 2026-04-30
- [20260430-002 弹框蒙层(overlay)遮挡弹框内容(content)](./ui-ux/20260430-002-overlay-z-index-stacking/report.md) — 2026-04-30
- [20260430-001 管理端内容区布局宽度被错误限制为 max-w-[1200px]](./ui-ux/20260430-001-layout-max-width-constraint/report.md) — 2026-04-30

## workflow-flow

- [20260516-002 技能配置流程缺少步骤化向导引导，用户可跳过关键步骤直接启用](./workflow-flow/20260516-002-skill-config-wizard-flow/report.md) — 2026-05-16 — **High**
- [20260514-004 Preferences API 500 — 跨 schema 查询架构设计缺陷](./workflow-flow/20260514-004-schema-fetch-architecture/report.md) — 2026-05-14 — **High**
- [20260508-002 管理端页面代码放置位置错误](./workflow-flow/20260508-002-admin-code-wrong-location/report.md) — 2026-05-08 — **High**
- [20260506-004 API 详情页编辑/禁用按钮不生效 + 管理端全局按钮功能审计](./workflow-flow/20260506-004-api-detail-buttons-not-working/report.md) — 2026-05-06
- [20260425-001 Design Fidelity Gate 未真正执行](./workflow-flow/20260425-001-design-fidelity-gate-not-executed/report.md) — 2026-04-25
