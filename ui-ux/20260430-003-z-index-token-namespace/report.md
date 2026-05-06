# Bug Analysis Report

---

## Metadata

| Field | Value |
|-------|-------|
| **ID** | `20260430-003` |
| **Title** | z-index token 命名空间错误导致 @theme 定义无效，弹框仍被蒙层遮挡 |
| **Type** | `ui-ux` — 界面/体验问题（前次修复 20260430-002 的回归） |
| **Severity** | High |
| **Status** | open |
| **Analyzed At** | 2026-04-30 |
| **Project** | diy-a2ui-master |
| **Version** | v1.2 |
| **Related Bug** | `20260430-002` — 上次修复引入了新的根因 |

---

## 1. Bug Description

### 1.1 Summary

上一次 bug 分析（20260430-002）修复了"弹框蒙层遮挡内容"问题，方案是建立 z-index token 体系。修复后用户反馈：**弹框仍然被蒙层遮挡**，问题未解决。

### 1.2 Reproduction Steps

1. 启动 admin-frontend 开发服务器
2. 导航到环境管理页面
3. 点击"编辑"按钮触发 EnvironmentForm Dialog
4. 观察：屏幕变为半透明黑色蒙层，弹框内容被遮挡
5. 打开 DevTools → Elements → 检查蒙层 div 和弹框 div 的 computed style
6. 确认：两者的 `z-index` 均为 `auto`（即未设置）

### 1.3 Expected Behavior

蒙层 `z-index: 30`，弹框内容 `z-index: 40`，弹框在蒙层之上。

### 1.4 Actual Behavior

蒙层和弹框的 `z-index` 均为 `auto` —— Tailwind v4 编译后的 CSS 中**不存在** `.z-overlay` 和 `.z-modal` 工具类。

---

## 2. Root Cause Analysis（三层归因）

### 2.1 Immediate Cause（直接原因）

`@theme` 中的 z-index 变量命名空间写错了。

```css
/* ❌ 实际写的 — 错误命名空间 */
@theme {
  --z-overlay: 30;
  --z-modal: 40;
}

/* ✅ Tailwind v4 要求的 — 正确命名空间 */
@theme {
  --z-index-overlay: 30;
  --z-index-modal: 40;
}
```

**证据**：编译后的 CSS（`dist/assets/index-*.css`）中：
- 只有 `.z-50{z-index:50}` （Tailwind 默认）
- 没有 `.z-overlay`、`.z-modal`、`.z-dropdown` 等工具类
- 没有任何 `--z-overlay`、`--z-modal` CSS 自定义属性

**Tailwind v4 命名空间规则**：`@theme` 中 `--z-index-*` 命名空间映射到 `z-*` 工具类前缀。

| `@theme` 变量 | 生成的工具类 | 生成的 CSS |
|---------------|-------------|-----------|
| `--z-index-overlay: 30` | `z-overlay` | `.z-overlay { z-index: 30 }` |
| `--z-overlay: 30` | **不生成** | **不存在** |

### 2.2 Root Cause（根本原因）

**开发者不熟悉 Tailwind v4 `@theme` 的命名空间规则，且缺乏验证手段。**

具体表现为：
1. **API 知识盲区** — `@theme` 中不同 CSS 属性有特定的命名空间前缀（`--color-*`、`--z-index-*`、`--spacing-*` 等），不是所有属性都直接映射到其 CSS 属性名
2. **修复后未验证** — commit message 写了"修复弹框蒙层遮挡内容问题"，但没有验证编译后的 CSS 是否正确生成了新的工具类
3. **无自动化检测** — 没有 CI 步骤或构建钩子来验证自定义 token 是否生效

### 2.3 Workflow-Level Cause（工作流层面的原因）

**为什么工作流允许"假修复"被提交？**

| 工作流缺陷 | 说明 |
|-----------|------|
| **修复后无验证** | 开发者修改了 `@theme` 定义但未检查编译产物，确认 token 真正生效 |
| **代码审查无法发现** | 审查者看到 `z-overlay` 类名和 `@theme` 定义，认为"逻辑正确"，无法肉眼判断编译结果 |
| **CI 无 CSS 产物校验** | 没有步骤验证"声称存在的 CSS 工具类确实出现在编译产物中" |
| **文档缺失** | 项目 CLAUDE.md 和 index.css 注释都没有记录 Tailwind v4 `@theme` 的命名空间映射规则 |
| **AI 辅助开发陷阱** | AI 生成的修复代码看起来合理（语义正确），但不一定符合 Tailwind v4 的具体 API 约定 |

---

## 3. Mechanism Analysis — 如何从机制上杜绝此类问题

### 3.1 问题分类：Framework API 知识错误 + 修复验证缺失

这是一类典型的 **"修复看起来正确但实际无效"** 的 bug。其特征是：

1. **代码意图正确** — `@theme` 定义 z-index token 的思路完全正确
2. **API 用法错误** — 具体的变量命名不符合框架规范
3. **验证缺失** — 没有验证修复是否生效

### 3.2 防御机制设计

#### 机制 1：构建产物断言（Build Artifact Assertion）

**原理**：CSS token 定义后，必须验证编译产物中存在对应的工具类。

**实施方案**：在 `package.json` 中添加 postbuild 脚本：

```bash
# admin-frontend/scripts/verify-css-tokens.sh
# 验证 @theme 中定义的 token 在编译产物中存在对应的工具类

CSS_FILE=$(ls dist/assets/index-*.css | head -1)

# 验证 z-index token
for token in overlay modal dropdown base toast; do
  if ! grep -q "\.z-${token}" "$CSS_FILE"; then
    echo "ERROR: .z-${token} utility class not found in compiled CSS"
    echo "Check @theme --z-index-${token} definition in src/index.css"
    exit 1
  fi
done
echo "OK: All z-index tokens verified in compiled CSS"
```

#### 机制 2：@theme 命名空间文档化

**原理**：在项目的 CSS 入口文件中，用注释明确记录 Tailwind v4 的命名空间规则。

```css
/*
 * Tailwind v4 @theme 命名空间映射规则：
 *
 * CSS 属性       → @theme 变量前缀   → 工具类前缀
 * z-index       → --z-index-*       → z-*
 * color         → --color-*         → text-*, bg-*, border-*
 * spacing       → --spacing-*       → p-*, m-*, gap-*, etc.
 * font-size     → --text-*          → text-*
 * border-radius → --radius-*        → rounded-*
 * shadow        → --shadow-*        → shadow-*
 * animation     → --animate-*       → animate-*
 */
```

#### 机制 3：修复验证清单（Fix Verification Checklist）

**原理**：在 bug 修复的 commit 前增加验证步骤，防止"假修复"。

```
修复验证清单（Fix Verification Checklist）:
□ 代码修改后，重新构建项目
□ 检查编译产物中是否包含预期变更
□ 在浏览器中实际复测 bug 是否修复
□ 如果是 CSS/样式修改，用 DevTools 检查 computed style
```

#### 机制 4：CI 静态分析 — 检测"未使用的 token 定义"

**原理**：如果在 `@theme` 中定义了变量，但编译后的 CSS 中不存在对应的工具类，说明命名空间可能写错了。

### 3.3 对 AI 辅助开发的特殊意义

此类 bug 在 AI 辅助开发中更容易发生，因为：

1. **AI 训练数据可能包含旧版 API** — Tailwind v4 是重大更新，AI 可能混用 v3 和 v4 的 API
2. **AI 不执行代码** — AI 生成代码时无法验证编译结果
3. **"看起来正确"不等于"实际正确"** — 代码审查（人工或 AI）容易被表面逻辑误导

**防范措施**：
- AI 生成的 CSS/框架配置代码，必须经过编译验证
- 在 CLAUDE.md 中记录框架的关键 API 约定
- 建立"AI 生成 → 编译验证 → 实际测试"的三步验证流程

---

## 4. Fix Proposal

### 4.1 Fix Path

**Decision**: `plan` — 直接修改，影响文件 1 个（`index.css`），无需创建 story。

### 4.2 Fix Details

修改 `admin-frontend/src/index.css` 中的 `@theme` 块：

```css
/* Before（错误）*/
@theme {
  --z-base: 10;
  --z-dropdown: 20;
  --z-overlay: 30;
  --z-modal: 40;
  --z-toast: 50;
}

/* After（正确）*/
@theme {
  --z-index-base: 10;
  --z-index-dropdown: 20;
  --z-index-overlay: 30;
  --z-index-modal: 40;
  --z-index-toast: 50;
}
```

生成的工具类仍然是 `z-base`、`z-dropdown`、`z-overlay`、`z-modal`、`z-toast`，**组件代码无需修改**。

### 4.3 Verification

修复后执行：

```bash
cd admin-frontend
pnpm build
# 验证编译产物
grep -oP '\.z-(overlay|modal|dropdown|base|toast)\{[^}]*\}' dist/assets/index-*.css
# 预期输出：
# .z-base{z-index:10}
# .z-dropdown{z-index:20}
# .z-overlay{z-index:30}
# .z-modal{z-index:40}
# .z-toast{z-index:50}
```

---

## 5. Impact Assessment

| Area | Impact |
|------|--------|
| Workflows affected | 无工作流影响 |
| Files modified | `index.css`（仅 1 文件） |
| Tests required | 构建验证 + 浏览器实测 |
| Migration needed | 否 — 工具类名称不变 |
| Risk | 零 — 只修正 CSS 变量名，不改变组件代码 |

---

## 6. Lessons Learned

1. **框架 API 约定必须查阅官方文档** — `@theme` 的命名空间映射不是直觉可推的，必须查文档
2. **修复后必须验证编译产物** — "代码看起来正确" ≠ "实际生效"
3. **CSS token 体系需要编译验证** — 不能只看源码就判断"已修复"
4. **AI 生成代码需要额外验证层** — AI 的框架知识可能过时或不精确
5. **文档化框架关键 API 约定** — 在项目中记录命名空间规则，防止重复犯错

---

*Generated by `bmad-analyze-bug` workflow*
