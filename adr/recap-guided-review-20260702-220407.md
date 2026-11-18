# 回顾：引导式评审（ADR 006）

日期：2026-07-02 · 分支：`feat/guided-review`（未提交）

## 构建内容

引导式评审现已成为代码评审应用的一等功能：`guide` agent-job 提供者生成当前变更集的分章评审（核心实现在前，影响在后，胶水代码分离），以全屏接管方式渲染——文件树和中央停靠区通过 CSS 隐藏，头部和右侧边栏保持不变。每个章节将 Markdown 概览与可实时标注的逐文件 diff 配对，并带一个 Reviewed 复选框，勾选后折叠该章节（可窥视重新展开，且不会取消标记）。适用于 PR 和所有本地 diff 类型；存在 PR 时，引导标题取自 PR。标注对等性端到端成立：引导 diff 挂载真实的 `DiffViewer`，使用文件作用域处理器，落入相同的 `CodeAnnotation` 状态，并通过相同的 Send Feedback 载荷导出。

## 如何组合

**流程：** ADR → 4 项研究 spike（子代理 map-reduce）→ 综合 → 规格 → 预检（逐一亲手验证每个接缝；纠正了两项 spike 结论）→ 4 个实现阶段（子代理处理不相交的文件集）→ 对抗性自审（10 项发现）→ 13 项修复 → 构建通过。

**服务端**（`packages/server/guide/guide-review.ts`，镜像 tour 模块）：`GUIDE_REVIEW_PROMPT` + `GUIDE_SCHEMA_JSON`（真实文件引用——绝不伪造 hunk）、`buildGuideUserMessage` 带变更文件列表（来自 `packages/shared/review-core.ts` 中新增的 `listPatchFiles`）、Claude/Codex 命令构建器、防御性输出清理，以及 `createGuideSession()`——其摄取校验会丢弃伪造路径、去重（首次放置优先）、计算 `unplacedFiles` 使每个变更文件恰好出现一次，并在无内容存活时失败关闭（`GUIDE_EMPTY_OUTPUT_ERROR`）。通过现有接缝接入：`SERVER_BUILT_PROVIDERS`、`guide` 能力、`buildCommand`/`onJobComplete` 中的 `provider === "guide"` 分支（完成时针对*当前* patch 校验，与客户端解析引用的方式一致）、路由 `GET /api/guide/:jobId` + `PUT /api/guide/:jobId/reviewed`。Pi 镜像：`vendor.sh` 引入 `shared/guide.ts` + `guide-review.ts`；`serverReview.ts` 和 Pi 的 `agent-jobs.ts` 手工镜像分支、路由和提供者注册。

**客户端**（`packages/review-editor`）：`guideOpen` 接管分支（停靠区保持挂载——CSS `hidden`；两个树分支均受门控）、树切换旁的 Guide 胶囊徽章、`Mod+Shift+G`（在 `reviewEditorShortcuts` 中注册以供文档生成器使用）、完成时自动打开（tour 模式）、空状态中的失败任务错误横幅、引导打开时 `focusedFilePath` 置空，外加引导本地焦点仲裁器。`components/guide/`：`GuideScreen`（空 / 生成中 / 视图）、`GuideEmptyState`（基于 `guide*` 设置切片的 Notion 风格启动页）、`GuideGenerating`（实时日志 + 取消）、`GuideView`（+ 来自 `unplacedFiles` 的 "Everything else"）、`GuideSectionCard`（窥视语义）、`GuideDiffSection`（ReviewDiffPanel 形状的适配器，使用文件作用域处理器变体；文件缺失 → "outdated" 标签；`pendingSelection` 门控到聚焦的查看器）。`useGuideData` 复制 `useTourData`（防抖的 reviewed 持久化、keepalive 刷新、demo fixture 门控到 DEV）。共享复用是提取而非复制：`AgentControls.tsx`（选择器原语）、`renderMarkdownProse.tsx`（tour + guide 散文）、导出的模型目录。`useAgentSettings` 增加了 guide 切片和模块级跨实例同步（现有两个活跃消费者）。AgentsTab 增加了 Guided Review 模式 + "Open guide" 任务卡片操作，贯穿 ReviewSidebar。

## 验证

- `bun run typecheck`（所有包，重新引入 Pi）：通过。
- `bun test` packages/ui：308 通过，0 失败（包括新绑定的快捷键注册表覆盖）；Pi `server.test.ts`：12 通过。
- `apps/review` + `build:hook` 构建通过；在 `@source ./utils` 修复后，已验证 Tailwind 输出包含 callout 选择器。
- 自审确认：启动快照纪律、失败关闭摄取、逐任务 reviewed 状态、标注数据对等、停靠区永不卸载、`listPatchFiles` 路径保真（重命名/删除/带引号/工作区前缀）。

## 已知缺口（v1 接受）

- Guide 和 reviewed 状态按服务端进程驻留内存（tour 先例；持久化存储由 ADR 006 推迟）。
- 引导 diff 渲染在固定高度（420px）盒子中，内部滚动——`DiffViewer` 填充有界父容器；实际使用后重新审视尺寸。
- 侧边栏标注点击无法揭示已折叠章节的 diff；本轮引导查看器内无搜索/`lineStart` 滚动定位。
- 重新加载后 SSE 快照重放会触发自动打开（tour 对等行为）。
- Marker 引擎（cursor/opencode）不是 guide 生成器。

## 参考文件

- ADR：`adr/decisions/006-guided-review-first-class-feature-20260702-192821.md`
- Spikes：`adr/research/SPIKE-guide-{provider-tour-pattern,launch-settings-reuse,diff-annotation-reuse,takeover-layout}-20260702-194831.md`
- 综合：`adr/research/synthesis-guided-review-20260702-195351.md`
- 规格（经预检 + 修复更新）：`adr/specs/guided-review-20260702-195351.md`
- 新代码：`packages/shared/guide.ts`、`packages/server/guide/`、
  `packages/review-editor/components/guide/`、`packages/review-editor/hooks/guide/`、
  `packages/review-editor/demoGuide.ts`、
  `packages/review-editor/utils/renderMarkdownProse.tsx`、
  `packages/ui/components/AgentControls.tsx`