# SPIKE：packages/review-editor/App.tsx 中的引导式审查接管布局

日期：2026-07-02

## 问题

`App.tsx` 目前如何组合审查界面？按照 ADR 006 添加全屏"引导"接管（隐藏文件树和中央停靠区，但保留头部和（可选）右侧边栏）的最干净方式是什么——同时不干扰现有状态（标注、agent 任务、diff 切换机制）？

## 1. 渲染结构（App.tsx，约第 2250 行起）

顶层 return，`packages/review-editor/App.tsx:2250-2256`：

```
<ThemeProvider defaultTheme="dark">
  <TooltipProvider ...>
  <ReviewStateProvider value={reviewStateValue}>
  <JobLogsProvider value={jobLogsValue}>
  {isSwitchingPRScope && <PRSwitchOverlay />}
  <div className="h-screen flex flex-col bg-background overflow-hidden">
    <header>...</header>
    <div className="flex-1 flex overflow-hidden">...</div>  {/* 主行 */}
  </div>
```

- **头部**（`App.tsx:2258-2668`）：一个 flex 行，`justify-between`。左侧集群（`App.tsx:2259-2324`）依次包含：文件树切换按钮（仅 `shouldShowFileTree && ...`，`App.tsx:2260-2274`）、一条垂直分隔线，然后是 PR 集群（仓库标签 + `PRSelector` + `StackedPRLabel`，`App.tsx:2276-2302`）或分支/仓库标签（`App.tsx:2303-2320`），或演示模式下仅一个"Review"标签。右侧集群（`App.tsx:2326`+）包含目标下拉菜单、错误/通知胶囊，最后是 `ReviewHeaderMenu`（`App.tsx:2652-2666`）。**这正是"引导"徽章该放的位置**：作为左侧集群中的新元素，紧跟在文件树切换按钮的分隔线之后（`App.tsx:2274` 之后），这样从左到右读起来就是"导航控件，然后是内容标识"。
- **主内容行**（`App.tsx:2670-2871+`）：一个 flex 行内的三个条件兄弟节点——左侧面板（`SectionsPanel` 或 `FileTree`，由 `shouldShowFileTree && isFileTreeOpen` 控制）、中央停靠区（`<div className="flex-1 min-w-0 ...">`，`App.tsx:2793-2871`）和右侧边栏（`ReviewSidebar`，由 `reviewSidebar.isOpen` 控制，`App.tsx:2874`+）。引导接管是**同一层级的第四个兄弟节点/分支**——具体插入形态见 §7。
- `TourDialog`（`App.tsx:3125`）和 `PRSwitchOverlay`（`App.tsx:2255`）是现有的两个"全屏式"界面。两者都**不是 portal**（两个文件中都没有 `createPortal`）——它们是直接内联渲染在树中的普通子节点，纯靠 CSS 实现全视口：`PRSwitchOverlay.tsx:5` 使用 `fixed inset-0 z-[100]`；`TourDialog.tsx:199` 使用 `fixed inset-0 z-50 flex items-center justify-center`。两者都**浮在**下方未受影响的布局之上（对话框/背景遮罩模式）——文件树、停靠区和头部仍然挂载并渲染在它们后面，只是被视觉遮挡。
- **引导为什么不应照搬这种模式**：`fixed inset-0` 对话框适用于临时覆盖层（PR 切换加载动画、模态导览）。按照 ADR 006，引导是**主要阅读模式**——它替换工作区，而不是浮在其上。将其建模为 `fixed` 覆盖层会：(a) 让停靠区保持挂载，其 resize/scroll 监听器在下方无意义地存活；(b) 与头部自身的 `z-50` 冲突；(c) 让"引导是否是当前视图"变成 z-index 问题而非布局问题。正确的形态是**主内容行中的条件渲染分支**（就像文件树/停靠区/边栏兄弟节点那样），而不是堆叠覆盖层。

## 2. 状态归属、现有视图切换、停靠区持久化

- 视图切换标志已经作为普通 `useState` 存在于 `App.tsx` 中：`isFileTreeOpen`（`App.tsx:216`），以及通过 `useSidebar<ReviewSidebarTab>(false, 'annotations')` 的 `reviewSidebar`（`App.tsx:215`，Hook 位于 `packages/ui/hooks/useSidebar.ts`）。`useSidebar` 是通用且可复用的——它基于 `useState` 返回 `{ isOpen, activeTab, open, close, toggleTab }`，无持久化、无 context。一个 `guideOpen` 布尔值（或如果引导将来需要子标签页，复用 `useSidebar` 形态）是 `isFileTreeOpen` 的同级，在 `App.tsx:216` 附近声明。
- **建议使用单一的 `reviewView: 'dock' | 'guide'` 状态**（或普通布尔值 `guideOpen`），而不是在现有标志之上再加布尔标志——这让"屏幕上是什么"成为单一事实来源，而不是需要手动协调 `isFileTreeOpen`/停靠区可见性/`guideOpen`。切换到引导模式完全不需要触碰 `isFileTreeOpen` 或 `reviewSidebar` 状态——它只改变主内容行中受控的内容（见 §7）；它们的状态（文件树宽度、哪个边栏标签页）保持完整，以便用户返回停靠区视图时恢复。
- **键盘快捷键接线**：完全遵循现有约定。`toggleFileTree`（`Mod+B`）和 `toggleSidebar`（`Mod+.`）在 `reviewEditorShortcuts`（`packages/review-editor/shortcuts.ts:58-69`）中声明为 action，处理函数通过 `App.tsx` 调用处的 `useReviewEditorShortcuts({ handlers: { toggleFileTree: ... } })` 接线。`toggleGuide` action 属于同一作用域（`packages/review-editor/shortcuts.ts`，`section: 'Layout'`，与 `toggleFileTree`/`toggleSidebar`/`toggleTour` 并列）——注意 `toggleTour`（`Mod+Shift+T`，仅开发环境，`shortcuts.ts:70-76`）是"通过快捷键打开全接管式界面"最接近的先例。
- **DockviewReact 在卸载时会丢失布局吗？会，而且是决定性的——并且这已经是代码库中的关键依赖。** `App.tsx:2809-2818` 仅在 `files.length > 0` 时条件渲染 `<DockviewReact ... onReady={handleDockReady} />`；当 `files.length === 0` 时改为渲染空状态 div（`App.tsx:2818-2870`）。`handleDockReady`（`App.tsx:709-740`）调用 `setDockApi(event.api)` 并**在每次挂载时重新**接线 `onDidActivePanelChange`——没有恢复已保存的布局。一个单独的 effect（`App.tsx:876-883`，由 `needsInitialDiffPanel.current` 控制）每次挂载只重新添加一次初始 diff 面板。所以今天，从"有文件"→"无文件"→"又有文件"（例如切换到空 diff 类型再切回）**已经会丢弃整个停靠区布局**（打开的标签页、分割面板、哪个面板处于活动状态）并重建单个默认面板。这对该特定转换是可接受的行为，因为它与真实内容变化绑定（无文件 = 无内容可显示）。**这对引导接管是不可接受的行为**，用户期望返回时精确恢复离开时的停靠区状态。结论：引导分支必须**用 CSS 隐藏停靠区的包装器**，而不是像空文件树状态那样条件卸载 `<DockviewReact>`。

## 3. 头部徽章约定

- 文件树切换按钮，精确代码：`App.tsx:2262-2272`：

```tsx
<button
  onClick={() => setIsFileTreeOpen(prev => !prev)}
  className={`p-1 rounded-md transition-all focus-visible:outline-none ${
    isFileTreeOpen
      ? 'text-primary'
      : 'text-muted-foreground hover:text-foreground hover:bg-muted'
  }`}
  title={isFileTreeOpen ? 'Hide file tree' : 'Show file tree'}
>
  <FolderTree className="w-3.5 h-3.5" />
</button>
```
紧随其后的分隔线：`<div className="w-px h-5 bg-border/50 mx-1 hidden md:block" />`（`App.tsx:2273`）。
- 目前没有与 ADR 006 描述完全一致的头部"胶囊/徽章按钮"组件（带标签的可点击徽章，而不仅是图标按钮）。可借鉴的最接近约定：
  - **目标下拉按钮**（`App.tsx:2334-2348`）：`flex items-center gap-1 px-2 py-1 rounded-md text-xs font-medium bg-muted hover:bg-muted/80 transition-colors`——这是带图标 + 文本的标签胶囊按钮的正确形态，通过背景色调区分活动/非活动。
  - **`CountBadge`**（`packages/review-editor/components/CountBadge.tsx`）——"紧凑等宽计数徽章——在文件树、标注和 AI 标签页中保持一致"，用于数字计数，不用于标签。
  - **`ConventionalLabelBadge`**（`packages/review-editor/components/ConventionalLabelPicker.tsx:139-152`）——一个小型 `cc-inline-badge cc-tone-{tone}` 胶囊，但它是静态标签，不可交互。
  - 菜单项键盘提示在 `ActionMenuItem` 的 `badge` prop 内使用 `<KbdHint>`（`ReviewHeaderMenu.tsx:163,172`）——与头部徽章的视觉样式无关，但确认了 `badge` 作为此代码库中槽位命名的约定。
  - **建议**：引导徽章参照目标下拉按钮的类（胶囊、`text-xs font-medium`、muted 背景、hover 状态），活动/非活动处理参照文件树切换按钮（引导打开时 `text-primary`，否则 `text-muted-foreground`），放在 `App.tsx:2274` 文件树切换按钮分隔线之后。

## 4. ReviewStateContext + JobLogsProvider 访问

两个 provider 都包裹整个屏幕，位于头部/内容 div **之上**（`App.tsx:2253-2254`，在 `App.tsx:3149` 闭合），因此新引导分支中渲染的任何内容——由于它是同一树中更深的兄弟节点，而非独立挂载——都可通过 `useReviewState()`（`packages/review-editor/dock/ReviewStateContext.tsx:184`）和 `useJobLogs()`（`packages/review-editor/dock/JobLogsContext.tsx:21`）完全访问。`ReviewState` 已携带引导页面所需的一切：`allAnnotations`、`onAddAnnotation`/`onSelectAnnotation`/`onNavigateToAnnotation`（diff 标注对等，ADR 006 §5）、`files`/`rawPatch`、`agentJobs`、`aiMessages`/`onAskAI`、`prMetadata`。`useAgentJobs()` 本身是在 `App.tsx` 中调用一次的 Hook（其结果供给 `reviewStateValue.agentJobs` 和 `jobLogsValue`）——引导界面不需要自己的订阅，它读取与 `TourDialog` 已读取的相同 context 切片（`TourDialog.tsx:5,` `useReviewState()`）。无需新 provider；`TourDialog` 作为今天消费此 context 的全屏式界面的可用先例，已确认这一点。

## 5. 持久化约定

- **Cookie 支持的设置**：`packages/ui/config/settings.ts` 将每个设置声明为 `SettingDef`（默认值、`fromCookie`/`toCookie`、可选的 `serverKey`/`fromServer`/`toServer`），注册在 `SETTINGS` 对象中，并通过 `configStore` 单例（`packages/ui/config/configStore.ts`）经 `useConfigValue`/`configStore.get`/`configStore.set` 解析。`reviewPanelView`（`settings.ts:60-68`）是最接近的类比——仅 cookie（`serverKey: undefined`），因为它是纯 UI 布局偏好，服务器无需知道。**`guideSeen` 标志或"上次视图"偏好（停靠区 vs. 引导）正是这种形态**：仅 cookie 的 `SettingDef`，例如 `storage.getItem('plannotator-guide-seen')`，除非未来需求希望服务器在加载时预选引导视图，否则无需服务器同步。
- 原始 cookie 辅助函数位于 `packages/ui/utils/storage.ts`（`getItem`/`setItem`/`removeItem`），刻意基于 cookie 而非 localStorage，因为每个会话运行在随机端口上，而 cookie 按域而非按端口作用域（`storage.ts:1-8`）。
- **服务器端按会话持久化**：审查服务器通过 `contentHash(rawPatch)`（`draftKey`，`packages/server/review.ts:202`，在 diff 切换时于 `review.ts:1210,1251,1375,1488` 重新计算）经 `/api/draft`（`review.ts:1848-1851`）为草稿设键。Tour 功能的清单持久化（`PUT /api/tour/:jobId/checklist`，ADR 006 §"Reviewed checkbox"先例）更简单：`packages/server/tour/tour-review.ts:534-543,594,600`——一个按 job id 设键的内存 `Map<string, boolean[]>`（`tourChecklists`），无磁盘持久化，服务器重启即丢失。这是 ADR 006 中为引导"Reviewed"复选框明确命名的先例。如果引导视图状态（你在哪一页、哪些页标记为已审查）需要在同一服务器会话内跨页面重载存活，同样的按 job id 设键的内存 Map 模式适用；ADR 006 明确将引导持久化超出服务器内存的部分推迟为后续决策，所以现在构建任何更持久的东西都属范围蔓延。

## 6. 服务器路由约定 + Pi 镜像

- **`packages/server/review.ts`** 在请求处理器内用扁平、有序的 `if (url.pathname === "..." && req.method === "...")` 链匹配路由（例如 `review.ts:1037,1087,1103,1131,1162,1167,...,2039,2065`），正则仅用于路径参数：Tour 的两条路由正是先例——
  - `GET /api/tour/:jobId` → `url.pathname.match(/^\/api\/tour\/[^/]+$/)`（`review.ts:1016-1021`）
  - `PUT /api/tour/:jobId/checklist` → `url.pathname.match(/^\/api\/tour\/([^/]+)\/checklist$/)`（`review.ts:1024-1029`）
  引导任务结果端点（`GET /api/guide/:jobId`）和"标记页面已审查"端点（`PUT /api/guide/:jobId/checklist` 或类似）将遵循完全相同的形态，添加在 Tour 块旁边（`review.ts:~1015-1030`）。
- **`apps/pi-extension/server/serverReview.ts`** 是**手工维护**的，不是生成的——由 `apps/pi-extension/vendor.sh` 确认。Vendoring 仅将**业务逻辑模块**逐字复制（带 import 路径重写）到 `apps/pi-extension/generated/`：`packages/shared/*.ts`（一个大的允许列表，`vendor.sh:9`）、选定的 `packages/server/*.ts` 模块（`agent-review-message`、`codex-review`、`claude-review`、`review-findings`、`marker-review`、`path-utils`、`review-skill-loader`，`vendor.sh:15-25`），以及特别地 `packages/server/tour/tour-review.ts`（`vendor.sh:28-36`，由 `serverReview.ts:103` 作为 `../generated/tour-review.js` 导入）。**HTTP 路由接线本身不被 vendored**——`serverReview.ts` 使用 `node:http` 原语手工重新实现相同的路由匹配（`serverReview.ts:960-977` 的正则匹配路径，几乎逐行镜像 `review.ts` 的 Tour 路由）。CLAUDE.md 的"两个实现都必须更新"在这里字面适用：添加引导路由意味着 (a) 编写 `packages/server/guide/guide-review.ts`（镜像 `packages/server/tour/tour-review.ts` 的形态——任务 provider、schema、解析器、内存结果存储），(b) 将其添加到 `vendor.sh` 的 vendored 文件列表（在 `tour-review` 块的模式中加一行，`vendor.sh:28-36`），以及 (c) 在 `serverReview.ts` 中现有 Tour 块旁边（`serverReview.ts:959-977`）手工添加匹配的 `if (url.pathname.match(...))` 块。

## 7. 对引导接管的影响

**状态**：一个标志拥有"引导是否打开"——普通 `useState<boolean>`（`guideOpen`）或 `useState<'dock'|'guide'>('view')`，在 `isFileTreeOpen`（`App.tsx:216`）旁边声明。它不需要叠加到 `isFileTreeOpen`/`reviewSidebar` 上——那些在引导打开时保持自身状态不变，因此返回停靠区视图时精确恢复原状。仅当产品希望空状态落地页记住首次 vs. 回访状态时，才通过 `packages/ui/config