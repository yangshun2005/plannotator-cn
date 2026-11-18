# SPIKE：为 Guide 空状态复用 agent-job 启动 UI + 设置机制

日期：2026-07-02

## 问题

ADR 006（Guided Review）称，空状态的启动控件复用与 Agents 标签页“完全相同的 agent-job 设置机制”，通过现有 agent-jobs 基础设施以新的 `guide` provider 启动。究竟哪些可复用、哪些与 Agents 标签页的 dock 上下文耦合、一个从零构建的 guide 启动界面实际需要接入什么？本 spike 端到端记录启动 UI + 设置栈，以便 guide 空状态实现能引用确切的调用点，而不必重新推导。

## 1. `AgentsTab.tsx` —— 模式、设置、能力、启动、任务列表

文件：`packages/ui/components/AgentsTab.tsx`（1122 行）。

**模式选择。** `AgentMode = 'review' | 'tour'`（来自 `useAgentSettings`，定义于 `packages/ui/hooks/useAgentSettings.ts:44`）。`availableModes`（第 685 行）由实时能力推导：`review` 当且仅当 `availableReviewEngines.length > 0` 时提供；`tour` 当且仅当 `tourAvailable && availableEngines.length > 0` 时提供。当可用模式 >1 时渲染为 `SelectMenu`，否则为静态标签（第 947-960 行）。guide 的“模式”会以同样方式插入——第三个条目，由 `guide` 能力门控。

**引擎/模型/effort 选择器并非独立组件——它们是大型 return 块（第 937-1121 行）中的内联 JSX，由同一文件前面定义的小型共享原语构建：**
- `ConfigRow`（216）——标签+控件行，`stacked` 变体用于全宽控件。
- `SegmentedPicker`（234）——用于小选项集（effort/reasoning）的胶囊选择器。
- `Toggle`（257）——开关（Codex 快速模式）。
- `SelectMenu`（278）——下拉+弹出层，用于引擎-模型和 review-profile 选择器。
- `renderEngineSelect<E>`（871）——通用图标按钮行，由引擎列表 + 图标/标签映射参数化；同时用于 Tour 的窄集（claude/codex）和 Review 的宽集（claude/codex/cursor/opencode）。
- `renderMarkerEngineConfig`（917）——共享的 Cursor/OpenCode 仅模型配置块。

这些都没有被抽成独立的 `<LaunchControls>` 组件——它们根据 `selectedMode` / `reviewEngine` / `tourEngine` 条件内联渲染。**模型/选项目录也是本文件的模块级常量**（`CLAUDE_MODELS`、`CLAUDE_EFFORT`、`CODEX_MODELS`、`CODEX_REASONING`、`TOUR_CLAUDE_MODELS`、`CURSOR_MODELS`、`OPENCODE_MODELS`，第 30-92 行）——并非从 `useAgentSettings` 导入，也未在别处共享。任何第二个启动界面要么从 `AgentsTab.tsx` 重新导入这些常量（它们目前未导出），要么复制它们。

**`useAgentSettings`** —— `packages/ui/hooks/useAgentSettings.ts`。通过 cookie 持久化（`getItem`/`setItem` 来自 `packages/ui/utils/storage.ts`，键 `plannotator.agents`，第 4 行），*而非*服务端配置——与 CLAUDE.md 的 Settings Persistence 一节所指出相同的基于 cookie 的模式（每会话随机端口排除了 localStorage）。状态形状（`AgentSettingsState`，第 50 行）：`selectedMode`、`reviewEngine`、`reviewProfileByEngine`（按引擎，非全局——第 55 行注释解释了原因）、`tourEngine`，以及按引擎的分区（`claude`/`codex`/`cursor`/`opencode`/`tourClaude`/`tourCodex`），每个含一个 `model` 外加一个 `perModel` 映射用于模型作用域的子设置（effort/reasoning/fast）——因此切换模型会记住该模型上次的 effort 选择（`patchClaude`/`patchCodex`，第 205-220、239-258 行）。该 hook 返回扁平化的 getter/setter（`claudeModel`、`claudeEffort`、`setClaudeModel`……）——这就是 ADR 所指的“设置机制”，它可独立复用且毫无障碍：`const settings = useAgentSettings()` 在任何组件中都能用，无需 context/provider（它只是普通 `useState` + 一个 `useEffect` cookie 同步，第 174-179 行）。

**能力获取/门控。** `AgentsTab` 自身不获取能力——它通过 prop 接收 `capabilities: AgentCapabilities | null`（来自 `useAgentJobs`，见 §2）。它通过扫描 `capabilities.providers` 推导出 `claudeAvailable`/`codexAvailable`/`tourAvailable`/`cursorAvailable`/`opencodeAvailable`（第 646-650 行）。两个协调 effect（702-724、730-744）在能力加载或变化时把已存选择校正回有效值（例如已保存引擎不再可用 → 回退到第一个可用项；已保存的 Cursor/OpenCode 模型 id 从实时目录中消失 → 重置为 `auto`/`''`）。

**`buildReviewLaunch`（780-814）/ `buildTourLaunch`（815-823）** —— 纯函数，闭包捕获 `settings` + `reviewProfiles` 状态，各自返回一个 `AgentLaunchParams`（类型见 §2）。Review 的形状随引擎而异（claude：`model`+`effort`；cursor：仅当非 `auto` 时可选 `model`；opencode：仅当非空时可选 `model`；codex：`model`+`reasoningEffort`+可选 `fastMode`），且四者都拼接 `...review`——仅当选中非默认 profile 时才含 `{ reviewProfileId }`（第 783 行），因此服务端默认解析路径是通过省略来触发的，而非显式 `"builtin:default"` 字符串。Tour 的构建器完全没有 `reviewProfileId`——§4 已确认。

**`profilesLoaded` 门控（629、825-833）。** 在 `/api/agents/review-profiles` 解析完成前，*自定义* profile 选择无法做启动校验（该 id 可能指向已被移除的 skill），因此 `canLaunch`（829）对非默认 `reviewProfileId` 保持按钮禁用，直到 `profilesLoaded` 为 true。Default 选择无需解析，从不等待。如果 guide 启动界面将来也允许用户选择自定义“guide 风格” profile，同样的门控逻辑需要复制（或抽取）——但据 §4，如今的 guide 启动根本不涉及 profile，因此该门控对 v1 无关紧要。

**任务列表渲染** —— `JobCard`（458-539）：状态经 `StatusSquare`（按 `AgentJobInfo['status']` 的彩色方块 + lucide 图标，`JOB_STATUS_BG`/`JOB_STATUS_ICON` 映射在第 159-173 行），已用时间经 `ElapsedTime`（148-155，一个 `setInterval` 重渲染计时器）或终态后静态 `formatDuration(endedAt - startedAt)`，注解数量徽章由 `externalAnnotations` 按 `ann.source` 分组（`annotationCounts`，747-755）并与 `job.source`（`AgentJobInfo` 上的 `"agent-{idPrefix}"` 字符串）匹配计算。`PendingLaunchCard`（541-568）在点击与服务器 201 响应之间渲染一张乐观的“starting”卡片。

## 2. `useAgentJobs.ts` —— 传输 hook

文件：`packages/ui/hooks/useAgentJobs.ts`。完全与服务器传输无关，可在任何组件中复用（`useAgentJobs(options?: { enabled?: boolean })`）。

**`AgentLaunchParams`**（23-33）—— 线上契约，原样以 JSON POST 到 `/api/agents/jobs`：
```ts
type AgentLaunchParams = {
  provider?: string; command?: string[]; label?: string;
  engine?: string; model?: string; reasoningEffort?: string;
  effort?: string; fastMode?: boolean; reviewProfileId?: string;
};
```
`launchJob`（225-242）执行 POST，从响应解析 `{ job: AgentJobInfo }`（`parseLaunchJob`，78-82），乐观地将其 upsert 进本地 `jobs` 状态，并在非 2xx 时以面向用户的消息拒绝（`readResponseError`，65-76）——`AgentsTab.handleLaunch`（835-857）将其捕获到 `launchError`。

**SSE 消费**（123-223）：挂载时打开 `EventSource('/api/agents/jobs/stream')`，处理 `snapshot` / `job:started` / `job:updated` / `job:completed` / `job:log` / `jobs:cleared` 事件类型（`AgentJobEvent` 联合，`packages/shared/agent-jobs.ts`）。`job:log` 事件追加到 `Map<jobId, string>`（`jobLogs`）而非替换——流式 stdout 增量。在 `onerror` 时，若从未收到 snapshot，则回退到按版本门控的轮询（`GET /api/agents/jobs?since=N`，未变时 304）——否则 SSE 重连交由浏览器原生重试。

**`killJob(id)`** → `DELETE /api/agents/jobs/:id`；**`killAll()`** → `DELETE /api/agents/jobs`。两者都是即发即忘（吞掉错误——“SSE 会协调”，249/258）。

返回形状：`{ jobs, jobLogs, capabilities, launchJob, killJob, killAll }`——guide 启动界面要消费的正是整个对象，且它与 AgentsTab 消费的是*同一个*对象（见 §6——只有一个实例）。

## 3. `/api/agents/capabilities` + 惰性 marker 模型目录

服务端：`packages/server/agent-jobs.ts`。处理器工厂 `createAgentJobHandler`（144）。二进制在**构造时探测一次**（158-171，`Bun.which(...)`）——`claude`、`codex`、`tour`（只要 claude 或 codex 二进制存在即可用），外加每个 `MARKER_ENGINES` 值一个条目（cursor、opencode——cursor 的二进制实际是 `agent`，第 169 行注释），并额外由 `mode === "review"` 门控（marker 引擎仅限 review，在 plan/annotate 模式下不可用）。

Marker（Cursor/OpenCode）**模型目录不在构造时探测**——`discoverMarkerModels(engine)`（128-142）以 5s 超时 spawn `<binary> <modelsArgv>`，仅在 `buildCapabilitiesResponse()`（174-186）内部调用，而后者仅在首次 `GET /api/agents/capabilities` 请求时调用，并缓存在 `markerModelsCache`（173）中，因此每个引擎每服务器生命周期只运行一次。这明确是为了避免 review-server 启动被缓慢/未认证的 CLI spawn 阻塞（第 119-127 行注释）。响应形状：`{ mode, providers: AgentCapability[], available: boolean }`，每个 provider 为 `{ id, name, available, models? }`。

## 4. 启动流程中的 review profile

`GET /api/agents/review-profiles`（`packages/server/review.ts:1798-1817`）返回 `{ profiles: [{ id, label, source, default? }, ...] }`——始终先包含 `BUILTIN_DEFAULT_PROFILE`（`id: 'builtin:default'`），然后每个精选 skill 一个条目（`discoverCuratedSkills()`，`id: 'skill:{name}'`，`source: 'user'`）。客户端在挂载时以及 `AddReviewDialog` 启用新 skill 后重新获取（`refreshReviewProfiles`，`AgentsTab.tsx:632-644`）。

客户端 → 服务端流程：`AgentsTab` 仅在 review 模式启动时发送 `reviewProfileId`，且仅当它不是默认值时（`buildReviewLaunch`，第 783 行）。服务端（`agent-jobs.ts:597`）将 `body.reviewProfileId` 传入传给 `buildCommand` 的 `config.reviewProfileId`。在 `review.ts:buildCommand`（621）中，`resolveRequestedReviewProfile(requestedProfileId)`（637，来自 `./review-skill-loader`）在 **`buildCommand` 顶部无条件调用，先于按 provider 分支**——包括 `provider === "tour"` 的情况（690-699）。这意味着 tour 启动的 `reviewProfile` 总是解析为内置默认值（Tour 从不发送 `reviewProfileId`，`buildTourLaunch` 已确认，`AgentsTab.tsx:815-823`，其中没有 `review` 展开），解析出的 `{ id: 'builtin:default', label: 'Default' }` 纯粹作为任务元数据附着在 tour 任务的 `reviewProfileId`/`reviewProfileLabel` 字段上（699）——**它从未被 `tour.buildCommand` 消费**（691-698，接收 `cwd`/`patch`/`diffType`/`options`/`prMetadata`/`config`——没有 `reviewProfile` 参数）。因此：tour 启动在结构上跳过 profile（解析会发生，但只是一个未被用于塑造 prompt 的空操作默认值）；遵循 Tour 先例的 guide provider 会做同样的事——走同一条 `buildCommand` 路径，解析出一个 profile 并盖在任务上以保持一致，但在构建自己的 prompt 时忽略它。

## 5. AgentsTab 引擎/模型/effort 选择器的可抽取性

今天不是组件——内联 JSX + 本地原语（`ConfigRow`、`SegmentedPicker`、`Toggle`、`SelectMenu`、`renderEngineSelect`、`renderMarkerEngineConfig`）全部定义在 `AgentsTab.tsx` 自身内，均未导出。**guide 空状态的最小复用并不需要抽取共享组件。** 两个现实选项：

**A. 直接调用 `useAgentSettings()` + 手写 guide 专用选择器（v1 推荐）。** guide 启动界面几乎肯定只需要一个引擎选择器（claude/codex）+ 模型 + effort——这是 Tour 已渲染内容的严格子集（Tour 的配置块，`AgentsTab.tsx:1010-1039`，几乎就是这个确切形状：`renderEngineSelect` + 模型 `SelectMenu` + effort `SegmentedPicker`/reasoning+fast）。`useAgentSettings` 需要一个小补充——一个 `guideEngine`/`guideClaude`/`guideCodex` 切片，镜像 `tourEngine`/`tourClaude`/`tourCodex`（hook 中第 57、62-63 行）——新 cookie 字段，增量式，无迁移风险。guide 界面随后导入 `useAgentSettings`，并通过字面复制 `ConfigRow`/`SegmentedPicker`/`SelectMenu`（它们很小，各约 20-60 行，除 `cn` 和 lucide 图标外无外部依赖）用*相同视觉原语*重新实现约 30 行 JSX；或者更好，只把这四个原语提升到共享文件（例如 `packages/ui/components/agentLaunchControls.tsx`），由 `AgentsTab.tsx` 和新 guide 界面共同导入——先小重构再复用，而非复制。

**B. 从 AgentsTab 抽取完整的 `<LaunchControls mode="guide">` 组件。** 更高保真的复用但更具侵入性：需要将 `selectedMode`/引擎可用性逻辑泛化到第三种模式，把 `guideEngine`/模型/effort 状态穿过同一条件渲染树，并有把接管屏样式（据 ADR 006 是“干净、优雅、Notion 风格”，而非 AgentsTab 使用的密集侧边栏标签密度）与 AgentsTab 的 Tailwind 类（`text-[9px]`、`text-[11px]` 等——为窄侧边栏调校，对全页接管屏是错误的尺度）耦合的风险。鉴于 ADR 明确希望空状态采用不同的视觉处理，此路径与设计目标相悖，只为边际的代码复用。

**建议：A**，只提升四个小型无状态原语，而非模式/引擎编排逻辑——guide 界面有更简单的决策树（若 guide 初期只在 `claude` 上发布，可能根本不需要引擎选择，参照 Tour 自身的初始引擎先例），不应继承 AgentsTab 的 review/tour 双模式分支。

## 6. 从另一界面观察任务完成/状态

**代码库中恰好只有一个 `useAgentJobs()` 调用点**——`packages/review-editor/App.tsx:350`（经全仓库 grep 确认）。AgentsTab **不**挂载自己的 hook；它通过 prop 接收 `jobs`/`capabilities`（`ReviewSidebar.tsx:537` 从 `App.tsx:2915-2919` 透传，而后者本身从单一 `agentJobs` 对象获取）。**在第二个组件中重复挂载 `useAgentJobs` 会向 `/api/agents/jobs/stream` 打开第二个 `EventSource`**——该 hook 自身没有去重/context；若两个组件各自调用它，没有任何机制阻止两条独立 SSE 连接和两份独立 snapshot。如果 guide 界面天真地自己调用 `useAgentJobs()`，这是真实风险。

如今状态扇出使用两个通道，而非一个：
1. **常规 React Context**——`ReviewStateContext`（或类似；`agentJobs: agentJobs.jobs` 在 `App.tsx:1854` 被展开进一个 context 值对象）把低频