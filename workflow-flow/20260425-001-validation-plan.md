# Validation Plan — Design Fidelity Gate 自动化

## 目标

将 Design Fidelity Gate 从"markdown 文档指令"升级为"可执行 Playwright 脚本 + CI 强制阻断"。

## 验证方案

### 方案：Playwright 视觉回归测试

#### 1. 基础设施

- 在项目中创建 `e2e/design-fidelity/` 目录
- 每个有原型的页面对应一个 `.spec.ts` 文件
- 使用 `toHaveScreenshot()` 做像素级对比

#### 2. 工作流

```
原型 HTML → Playwright 渲染 → 截图（golden image）
实现页面 → Playwright 渲染 → 截图
diff 对比 → 零容忍（maxDiffPixelRatio: 0）
失败 → 退出码非零 → CI 红灯
```

#### 3. 配置

```typescript
// playwright.config.ts 扩展
{
  testDir: './e2e',
  projects: [
    {
      name: 'design-fidelity',
      testDir: './e2e/design-fidelity',
      use: { viewport: { width: 1440, height: 900 } },
    },
  ],
}
```

#### 4. Pipeline 集成

- `design-fidelity-gate.md` 中增加"执行脚本"指令
- 在 `step-03-story-loop.md` PHASE 5b.6 中调用 `pnpm exec playwright test --project=design-fidelity`
- `pipeline-state.yaml` 模板增加 `fidelity_result` 必填字段

## 验证清单

- [ ] Playwright 视觉回归测试可执行
- [ ] 原型截图和实现截图自动对比
- [ ] 差异超过阈值时 CI 红灯
- [ ] pipeline-state.yaml 包含 fidelity_result
- [ ] 失败时自动生成 diff 图片用于定位
