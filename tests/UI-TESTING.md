# UI 测试指南

本指南帮助你测试 Plannotator 中的 UI 变更。无论是新增功能还是修复缺陷，都请按以下步骤确保变更正常工作。

## 目录

1. [开发环境搭建](#development-setup)
2. [开发工作流](#development-workflow)
3. [快速测试指南](#quick-testing-guide)
4. [常见问题调试](#debugging-common-issues)
5. [决策控件手动检查清单](#decision-control-manual-checklist)
6. [WebMCP 手动检查清单](#webmcp-manual-checklist)
7. [终端工具公告手动检查清单](#terminal-tools-announcement-manual-checklist)
   - [Ask This Session 公告手动检查清单](#ask-this-session-announcement-manual-checklist)
8. [HTML 同级链接手动检查清单](#html-sibling-link-manual-checklist)
9. [跨文件标注手动检查清单](#cross-file-annotations-manual-checklist)

---

## 开发环境搭建

### 前置条件

- **Bun** - JavaScript 运行时和包管理器（[安装](https://bun.sh)）
- **Git** - 版本控制
- **现代浏览器** - Chrome、Firefox、Safari 或 Edge（最新版本）

### 安装

```bash
git clone https://github.com/backnotprop/plannotator.git
cd plannotator
bun install
```

### Monorepo 结构

项目采用 monorepo 结构：

- **`packages/`** - 共享代码
  - `ui/` - 可复用的 React 组件、Hook、工具函数
  - `server/` - 服务端实现（计划/审查服务器）
  - `editor/` - 计划审查应用逻辑
  - `review-editor/` - 代码审查应用逻辑

- **`apps/`** - 可部署应用
  - `hook/` - Claude Code 插件（计划审查）
  - `opencode-plugin/` - OpenCode 插件
  - `review/` - 独立审查应用
  - `portal/` - 分享门户（share.plannotator.ai）
  - `marketing/` - 营销站点（plannotator.ai）

### 首次构建测试

验证环境是否正常：

```bash
bun run build:hook
```

成功后会生成 `apps/hook/dist/index.html`。

---

## 开发工作流

### 修改 UI

**共享组件**（计划 UI 和审查 UI 共用）：

- 位置：`packages/ui/components/`
- 示例：`TableOfContents.tsx`、`AnnotationToolbar.tsx`、`Viewer.tsx`

**计划编辑器**（计划审查 UI）：

- 位置：`packages/editor/App.tsx`
- 计划审查的主应用逻辑

**代码审查编辑器**（代码审查 UI）：

- 位置：`packages/review-editor/App.tsx`
- 代码审查的主应用逻辑

**工具函数和 Hook**：

- 位置：`packages/ui/utils/`、`packages/ui/hooks/`
- 示例：`parser.ts`、`useActiveSection.ts`、`annotationHelpers.ts`

### 开发服务器（热重载）

需要快速迭代时，使用带热重载的开发服务器：

```bash
# 计划审查 UI（最常用）
bun run dev:hook
# 打开 http://localhost:5173

# 代码审查 UI
bun run dev:review
# 打开 http://localhost:5174

# 门户（share.plannotator.ai）
bun run dev:portal

# 营销站点（plannotator.ai）
bun run dev:marketing
```

**注意：** 开发服务器独立运行，不集成插件。修改后立即生效，无需重新构建。

### 构建用于测试

需要测试真实插件集成时：

```bash
# 构建计划审查 UI
bun run build:hook
# 输出：apps/hook/dist/index.html

# 构建代码审查 UI
bun run build:review
# 输出：apps/review/dist/index.html

# 构建 OpenCode 插件
bun run build:opencode
# 从 hook/review 的 dist 目录复制 HTML

# 构建全部
bun run build
# 执行 build:hook && build:opencode
```

### 构建注意事项

**OpenCode 插件会从 hook 和 review 的 dist 目录复制预构建的 HTML 文件。**

修改 UI 时：

✅ **正确：**

```bash
bun run build:hook && bun run build:opencode
```

❌ **错误：**

```bash
bun run build:opencode  # 使用的是上次构建的旧 HTML！
```

修改 UI 代码后，务必先重新构建 hook/review，再构建 OpenCode。

---

## 快速测试指南

### 测试脚本

UI 测试脚本在本地模拟插件行为：

```bash
# 计划审查 UI 测试
./tests/manual/local/test-hook.sh          # Claude Code 模拟
./tests/manual/local/test-hook-2.sh        # OpenCode 来源徽章测试
./tests/manual/local/test-codex-plan-review-e2e.sh  # 真实 Codex Stop-hook 端到端测试

# 代码审查 UI 测试
./tests/manual/local/test-opencode-review.sh  # 代码审查 UI 测试
```

### 各脚本的作用

**`test-hook.sh`**

1. 构建 hook 插件（`bun run build:hook`）
2. 传入示例计划 JSON（含标题、SQL/TypeScript 代码、检查清单）
3. 启动本地服务器
4. 打开浏览器显示计划审查 UI
5. 在终端打印批准/拒绝决策

**`test-hook-2.sh`**

1. 构建 hook 插件
2. 带 `opencode` 来源标志启动服务器
3. 验证 UI 中出现蓝色 "OpenCode" 徽章
4. 测试来源检测逻辑

**`test-opencode-review.sh`**

1. 构建审查应用（`bun run build:review`）
2. 带示例 git diff 启动审查服务器
3. 打开浏览器显示代码审查 UI
4. 验证 "OpenCode" 徽章 + 头部决策控件（零标注时显示 `Approve`，有标注后显示 `Send Feedback`——不是 "Copy Feedback"）
5. 测试反馈提交流程

**`test-codex-plan-review-e2e.sh`**

1. 构建 hook + review 应用（除非指定 `--skip-build`）
2. 创建一次性 `HOME` 和示例 git 仓库
3. 将你的 Codex 认证复制到一次性配置中
4. 启用 `hooks` 并注册指向本地 Plannotator 入口的 `Stop` hook
5. 运行真实的 `codex exec` 提示，仅返回一个 `<proposed_plan>` 块
6. 在产物目录中保留 rollout 日志、Plannotator 历史、计划文件和会话 URL

需要验证完整的 Codex 拒绝/修改/批准循环（而非模拟 hook 载荷）时，这是最佳测试工具。如需浏览器自动化，设置 `PLANNOTATOR_BROWSER=/usr/bin/true`，在一个终端保持脚本运行，在另一个终端用 Playwright 驱动打印出的会话 URL。

更多集成和工具测试脚本见 [tests/README.md](../tests/README.md)。

### 手动测试流程

1. **在 `packages/ui/` 或 `packages/editor/` 中修改代码**

2. **选择测试方式：**
   - **方式 A：** 开发服务器（快速迭代）
     ```bash
     bun run dev:hook
     ```
   - **方式 B：** 构建后用脚本测试（集成测试）
     ```bash
     bun run build:hook && ./tests/manual/local/test-hook.sh
     ```

3. **验证变更**是否正常工作

4. **测试响应式设计：**
   - 桌面（>1024px）：完整布局含 TOC
   - 平板（768-1024px）：隐藏 TOC
   - 移动端（<768px）：触控优化
   - 使用浏览器 DevTools（F12）→ 设备工具栏（Cmd+Shift+M / Ctrl+Shift+M）

5. **检查浏览器控制台**是否有错误：
   - 打开 DevTools（F12）
   - Console 标签页
   - 查找红色错误

6. **在多个浏览器上测试**（Chrome、Firefox、Safari、Edge）

---

## 常见问题调试

### 浏览器 DevTools

打开 DevTools 进行检查和调试：

- **Mac：** Cmd+Option+I
- **Windows/Linux：** F12 或 Ctrl+Shift+I

**常用标签页：**

- **Console：** JavaScript 错误和日志
- **Network：** 失败的请求、慢资源
- **Elements：** 检查 DOM 和 CSS
- **Performance：** 分析渲染性能
- **Memory：** 检查内存泄漏

**推荐扩展：**

- React DevTools - 检查组件树和 Props
- Redux DevTools - 如果使用 Redux（当前未使用）

### 常见问题与解决方案

#### 端口已被占用

**错误：**

```
Error: listen EADDRINUSE: address already in use :::5173
```

**解决方案：** 杀掉占用该端口的进程

**macOS/Linux：**

```bash
lsof -ti:5173 | xargs kill -9
```

**Windows：**

```powershell
netstat -ano | findstr :5173
taskkill /PID <pid> /F
```

#### 找不到模块

**错误：**

```
Error: Cannot find module '@plannotator/ui'
```

**解决方案：** 清理并重新安装依赖

```bash
rm -rf node_modules
bun install
```

#### 热重载不工作

**症状：** 保存文件后浏览器中无变化

**解决方案：**

1. 强制刷新浏览器：Cmd+Shift+R（Mac）或 Ctrl+Shift+R（Windows/Linux）
2. 重启开发服务器：Ctrl+C 后执行 `bun run dev:hook`
3. 清除浏览器缓存
4. 检查终端是否有错误

#### CSS 不生效

**症状：** Tailwind 类不工作或样式异常

**解决方案：**

1. 检查类名拼写（Tailwind 严格匹配）
2. 确认 Tailwind 配置包含你的文件路径
3. 尝试重新构建：`bun run build:hook`
4. 检查是否有其他 CSS 规则覆盖（用 DevTools Elements 标签页）
5. 确认使用了正确的响应式前缀（`sm:`、`md:`、`lg:`）

#### TypeScript/LSP 错误

**症状：** 编辑器显示红色波浪线，但代码能运行

**重要：** 本代码库中许多 LSP 错误是警告，不是阻塞项。

**解决方案：**

1. 优先修复你修改的文件中的错误
2. 运行 `bun run build` 查看实际编译错误
3. 现有文件可能有警告——这是正常的
4. 如果你的文件中出现新错误，请修复

**可忽略的常见 LSP 警告：**

- "Alternative text title element cannot be empty"（SVG 图标）
- "This hook does not specify its dependency"（已知）
- "Provide an explicit type prop for button"（现有代码）

#### 构建失败

**错误：**

```
Build failed with X errors
```

**解决方案：**

1. 仔细阅读错误信息（会显示文件和行号）
2. 检查变更中的语法错误
3. 确认导入正确
4. 运行 `bun install` 确保依赖最新
5. 检查文件路径是否正确（Linux/macOS 区分大小写）

### 查看日志

**服务器日志：**

- 检查运行 `bun` 的终端
- 服务器会打印请求和错误
- Hook 输出显示批准/拒绝决策

**浏览器日志：**

- DevTools → Console 标签页
- Network 标签页显示请求/响应详情
- 勾选 Preserve log 可在页面加载间保留日志

**测试脚本输出：**

- 测试脚本打印到终端
- 显示构建输出、服务器启动和 hook 决策
- 用 `echo` 语句向脚本添加调试输出

---

## 决策控件手动检查清单

非 CI。所有标注界面和审查头部共用一个自适应分体控件（`DecisionControl`）：一个正向主按钮（`Done` / `Approve` / `Send Feedback`）加一个下拉菜单，包含备选决策和就地备注编辑器。每个流程都要在两种状态下运行——零标注和 n 个标注——在桌面和真实手机上都要测（触控没有 `Mod+Enter`，这正是该控件要修复的回归类型）。

1. **标注，单文件**（`plannotator annotate notes.md`）。零标注时主按钮显示 `Done`；点击提交"无反馈"记录，终端打印该记录。下拉菜单 → `Done with a note…` 就地打开编辑器：`Enter` 插入换行，`Mod+Enter` 提交，`Escape` 返回菜单并保留草稿。添加一个标注：主按钮变为 `Send Feedback`，`Done, discard 1 annotation…` 弹出一次确认。
2. **标注，gate 模式**（`plannotator annotate notes.md --gate --json`）。零标注时主按钮为 `Approve`，发送 `/api/approve`（stdout 记录 `"approved"`；带 `--require-approval` 时仅批准才退出 `0`）；`Request changes…` 记录带标注的决策。`Approve with a note…` / `Approve with notes` 仅在会话声明支持批准备注时出现。
3. **标注，文件夹和 last**（`plannotator annotate docs/`、`plannotator last`）。同一控件，同一状态；在文件夹会话中于草稿中途切换文档，确认头部计数跟随会话的标注。
4. **HTML / 实时应用标注**（`plannotator annotate page.html`、`plannotator annotate http://localhost:<port>`）。打开下拉菜单，然后点击框内页面：弹出层关闭（iframe 获得焦点即关闭信号——没有父级 pointerdown）。
5. **审查，agent 模式**（`plannotator review`）。零标注时 `Approve`，标注后 `Send Feedback`；有标注仍要批准需两次点击（下拉菜单 → `Approve, discard n annotations…` → `Discard & approve`）。编辑器打开时，`Escape` 返回菜单，不折叠文件树、不关闭侧边栏；再按一次 `Escape` 关闭菜单；第三次执行应用自身的退出阶梯。在打开的丢弃确认框上按 `Mod+Enter` 只能触发对话框，绝不能二次提交。
6. **审查，平台（PR）模式**（`plannotator review <pr-url>`）。控件形状相同，无编辑器项：每个菜单操作都打开 `ReviewSubmissionDialog`。在自己的 PR 上，批准行置灰并显示"You can't approve your own PR/MR"原因，而 `Request changes…` / `Post comments, then…` 保持可用。
7. **紧凑/触控**（真实手机或 DevTools 设备模式，两个应用）。头部菜单在每个状态下都有可见的正向决策行；编辑器行打开备注对话框（`DecisionNoteDialog`），而非内联文本域。
8. **全局评论**（审查）。零标注时（空状态）可访问 "+ Global comment"，也可从侧边栏的 Global comments 区标题进入，还可从引导式审查头部的 "Global comment" 按钮进入；创建后头部控件变为 `Send Feedback`。

## HTML 界面头部手动检查清单

非 CI。在没有 `plannotator-html-chrome` cookie 的浏览器配置中运行 `plannotator annotate <file>.html`（全新配置，或清除该 cookie）。

1. **打开时工具隐藏。** 页面填满视口：左侧无侧边栏舌片标签，右上无评论/附件簇，加载过程中也不闪现。头部眼睛图标显示"隐藏"（eye-off）状态，并报告 `aria-pressed="true"`。
2. **眼睛图标是返回入口。** 点击它：舌片标签和簇出现，图标翻转，版本差异的 "Show changes" 控件（文件有上一版本时）也在该簇中。重新加载：工具仍显示——新 cookie 胜过隐藏默认值。再次隐藏并重新加载：仍隐藏。
3. **切换快捷键。** `Mod+Shift+A` 将界面切到 Interact（笔未按下，点击到达页面），再按一次重新激活——包括焦点在框内页面时，这是 Esc 无法撤销的情况。`Mod+Shift+X` 可从任一文档切换工具。打开评论草稿，然后按 `Mod+Shift+A`：草稿随解除激活一起关闭，如同 Esc。
4. **工具提示。** 悬停眼睛、笔和 Refresh：每个都在快捷键上方以键帽形式显示描述（眼睛为 ⌘⇧X / Ctrl+Shift+X，笔为 ⌘⇧A / Ctrl+Shift+A，Refresh 无键行），定位在按钮下方，明暗主题均如此。Tab 到每个控件：键盘聚焦时打开相同工具提示。其上不出现原生 `title` 框。
5. **紧凑/触控**（DevTools 设备模式）。无头部眼睛或笔；Options 菜单在全新会话中显示 "Show tools"，使用后变为 "Hide tools"。

## WebMCP 手动检查清单

非 CI。在启用 API 的 Chrome 或 Edge 中运行：`chrome://flags/#enable-webmcp-testing`，或用 `--enable-features=WebMCPTesting` 启动。使用全新配置，避免首次运行对话框和恢复的草稿干扰。Model Context Tool Inspector 扩展也能调用工具，但页面控制台就够了：`const tools = await document.modelContext.getTools()` 列出工具，`JSON.parse(await document.modelContext.executeTool(tools.find((t) => t.name === 'plannotator.read_document'), {}))` 调用一个。

流程之前，确认足迹规则：

- 加载 `plannotator annotate <file.md>` 且不做任何操作。列出六个 `plannotator.*` 工具，但头部无 "Agent" 标记、无横幅，`document.cookie` 中无 `plannotator-webmcp-tools` 条目。
- 在没有该 API 的浏览器中加载同一会话。页面无任何变化，Settings