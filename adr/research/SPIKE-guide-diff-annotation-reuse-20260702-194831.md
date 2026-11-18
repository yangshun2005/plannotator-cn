# SPIKE：复用 Pierre diff + 标注机制用于 Guide 接管区块

日期：2026-07-02
相关：`adr/decisions/006-guided-review-first-class-feature-20260702-192821.md`

## 1. `DiffViewer.tsx` — Props、耦合、独立挂载可行性

文件：`packages/review-editor/components/DiffViewer.tsx:136-238`

**Props 契约**（`DiffViewerProps`，第 136-189 行）：所需一切均通过 Props 传入 — `patch: string`、`filePath`、`oldPath?`、`status?`、`reviewBase?`（用于 `/api/file-content`）、`prUrl?`/`prDiffScope?`（仅用于草稿命名空间）、显示标志（`diffStyle`、`diffOverflow`、`diffIndicators`、`lineDiffType`、`disableLineNumbers`、`disableBackground`、`expandUnchanged`、`fontFamily`、`fontSize`）、标注集合 + 回调（`annotations`、`selectedAnnotationId`、`scrollTargetAnnotation`、`pendingSelection`、`onLineSelection`、`onAddAnnotation`、`onAddFileComment`、`onEditAnnotation`、`onSelectAnnotation`、`onDeleteAnnotation`）、已查看/已暂存 UI 状态、搜索 Props 和 AI Props。**`DiffViewer.tsx` 内部零 dock 导入、零 dockview 类型、零 `useReviewState()` 调用。** 它已经是一个纯粹的、由 Props 驱动的组件。

**Dock 耦合完全在包装器中**，不在组件内：`ReviewDiffPanel.tsx:1-123` 是一个约 110 行的适配器，读取 `useReviewState()` 和 dockview 的 `filePath` 参数，执行 `state.files.find(...)` 查找，计算 `fileAnnotations`/`aiMessagesForFile`/`searchMatchesForFile`，并将一切传入 `<DiffViewer>`（第 69-120 行）。这证实了 ADR 在 Consequences 中的说法："任何与 dock 布局的耦合……都会被抽离而非复制" — 对 `DiffViewer` 而言无需抽离任何东西；guide 区块可以直接从 `useReviewState()` 构建相同的 Prop 包，完全无需触碰 `ReviewDiffPanel`。

**文件内容获取**（第 309-352 行）：在 `[filePath, oldPath, reviewBase]` 变化时，请求 `/api/file-content?path=&oldPath=&base=`，然后用完整文件内容通过 `processFile()` 重新解析 patch，以便 hunk 可以展开（`augmentedDiff`，第 331 行的 useMemo）。如果请求失败、为空或内容不再与 patch 一致（`isContentConsistentWithPatch`，第 337 行），则回退到原始 patch 的 `fileDiff`（`getSingularPatch(patch)`，第 307 行）— 静默回退，无错误 UI。这完全自包含于每个挂载实例；N 个独立 `DiffViewer` 各自执行独立请求（实例间无去重，但 dock 的单实例面板本来也是如此）。

**独立挂载可行性：** 可以。`DiffViewer` 只需要 `patch` + `filePath`（+ 可选的 `oldPath`/`reviewBase` 用于展开）以及标注 Prop 包。它有自己的滚动容器（`OverlayScrollArea`，通过 `useOverlayViewport` 拥有自己的 `containerRef`/`viewport`）、自己的 `ToolbarHost` 实例（第 722 行）、自己的 `FileHeader`/`FileCommentBanner`。没有任何地方假设它在 dockview 内部或它是 DOM 中唯一的实例 — `isFocused` Prop 的存在正是为了仲裁跨实例状态（见 §3/§6）。

## 2. `AllFilesCodeView.tsx` 作为替代方案

文件：`packages/review-editor/components/AllFilesCodeView.tsx`（2054 行；分两遍阅读）。

这是基于 Pierre 虚拟化 `CodeView`（单实例、多个 `CodeViewItem`）构建的"所有文件在一个滚动容器中"的界面。从架构上，它对 guide 区块来说是错误的形态：

- **一个 `CodeView` 实例旨在拥有整个文件集**，非受控/一次性种子（`initialItems`），在文件集变化时通过 `fileSetKey` 重新挂载（第 497-541 行）。使用"每个 guide 区块一个 `CodeView`"会将这套重型机制（身份映射、`augmentRef`、滚动空闲延迟增强队列、worker 池门控）乘以 N 倍，而每个区块通常只有 1-5 个小文件 — 纯开销，没有虚拟化的收益（虚拟化在 50+ 文件时才有意义，而非 2-3 个）。
- **依赖 worker 池就绪**（`useIsWorkerPoolReadyOrDisabled`，第 438 行）和全局高亮 worker 主题同步 — 围绕"一个大界面"设计，而非许多小的独立实例。
- **其 Prop 表面是 dock/全文件特定的**：`registerCollapseAllToggle`/`onAllCollapsedChange`（dock 标签栏全部折叠按钮）、`isActive`（将 `[ ] / z / v / a / c / x` 键盘快捷键门控到"这是活动面板"）、`fileOrder: 'tree' | 'list'`（镜像当前活动的左侧面板）、`canStagePath`（git 状态面板特定的暂存门控）。这些都无法干净地映射到"散文区块中的几个 diff"。
- 每文件标注操作通过 `activeFilePath`（用户最后点击的文件）间接路由，而非直接按文件寻址（`onAddAnnotationForFile(filePath, ...)`，第 165-174 行，通过第 629 行的 `handleAddAnnotation` 路由，后者读取 `activeFilePath`）。这对"一个大列表"没问题，但对"guide 已知路径的 N 个独立小 diff"很别扭。

**结论：** `DiffViewer` 更合适。包含一个或多个 diff 的 guide 区块应**每个文件渲染一个 `DiffViewer`**，与 dock 的单文件面板完全一样 — 只是内联组合在 guide 页面的布局中，而非 dockview 面板内。guide 页面规模无需虚拟化；简单性和 Prop 局部性胜出。

## 3. Dock 之外的标注接线：`ToolbarHost` + `useAnnotationToolbar`

文件：`packages/review-editor/components/ToolbarHost.tsx`、`packages/review-editor/hooks/useAnnotationToolbar.ts`。

`ToolbarHost`（第 24-52 行为 Props）接收 `patch`、`filePath`、`isFocused`、`onLineSelection`、`onAddAnnotation`、`onEditAnnotation`，以及可选的 AI Props — 与 `DiffViewer` 已经传递的形态完全相同（`DiffViewer.tsx:722-735`）。它在内部拥有 `useAnnotationToolbar`（第 74 行），因此每次击键的工具栏状态不会重新渲染 diff；父级通过 `ToolbarHostHandle`（`handleLineSelectionEnd`、`handleTokenClick`、`startEdit`）与之通信。

**不是单例** — 它设计用于多个同时存在的实例。`useAnnotationToolbar.ts:47-48` 将 `draftStore` 和 `restoreDraftKeyByFilePath` 保持为**模块级 `Map`，以 `filePath` 为键**，而非组件/实例状态。这是有意为之：dockview 同时挂载单文件面板和全文件面板（它们是不同的 dockview 标签页，dockview-react 面板在 DOM 中保持存活而非在标签切换时销毁 — 见 `App.tsx:2810-2817` 的 `DockviewReact`，无 `disableTabsOverflowList`/keepalive 退出选项，且第 271-309 行现有的 `isFocused` 驱动的保存/恢复逻辑只有在同一文件的两个实例可以共存时才有意义）。`isFocused` Prop 正是仲裁此事的机制：在 `isFocused` true→false 时，当前范围的进行中草稿被暂存在 `restoreDraftKeyByFilePath.set(filePath, key)`（第 278 行）；在 false→true 时被重放（第 283-308 行）。**这意味着一个 guide `DiffViewer` 实例对应的文件同时在一个（现已隐藏的）dock 面板中打开，并不是一类新问题 — 这正是草稿系统已经存在要处理的多实例场景。** Guide 集成只需正确参与相同的 `isFocused` 契约（见 §6 的缺口）。

`renderAnnotation` / 内联渲染：`DiffViewer` 从 `annotations` 构建 `lineAnnotations`（过滤为 `scope ?? 'line' === 'line'`，通过 `lineAnnotationMetadata()` 映射为 Pierre 的 `{ side, lineNumber: ann.lineEnd, metadata }` 形态，第 510-518 行），合并 AI 标记（521-538），并将 `renderAnnotation`（552-577）+ `mergedAnnotations` 传入 `<FileDiff lineAnnotations=... renderAnnotation=...>`（通过 `PierreDiffContent`，第 107-109 行）。`renderAnnotation` 纯粹根据标注的 `metadata.kind` 分派到 `InlineAnnotation`（真实标注）或 `InlineAIMarker`（AI 标记）。整条路径是标注数组进、JSX 出 — 它没有 dock 依赖，除了 guide 区块已通过 `ReviewStateContext` 拥有的东西（`allAnnotations`，按 `filePath` 过滤）之外不需要任何东西。

## 4. `ReviewStateContext` — diff+标注路径实际需要什么，以及"免费"声明

文件：`packages/review-editor/dock/ReviewStateContext.tsx`。

已确认：`ReviewStateProvider` 在 `App.tsx:2253...3150` 中包裹了**整个**应用主体 — 头部、文件树、dockview 中央面板、侧边栏、`TourDialog`（挂载于第 3125 行，仍在 provider 内），以及任何未来的 guide 接管 — 全部都是 `<ReviewStateProvider value={reviewStateValue}>` 的后代。所以**是的**，在该树中任何位置作为兄弟/子节点渲染的 guide 接管屏幕，通过 `useReviewState()` 免费获得完整的 `ReviewState` 上下文 — 无需新上下文，无需 Prop 钻取。

diff+标注路径（`ReviewDiffPanel` + `DiffViewer` + `ToolbarHost`）实际消费的字段，全部已在 `ReviewState` 上：`files`、`reviewBase`、`activeDiffBase`、`diffStyle`/`diffOverflow`/`diffIndicators`/`lineDiffType`/`disableLineNumbers`/`disableBackground`/`expandUnchanged`/`fontFamily`/`fontSize`、`allAnnotations`、`selectedAnnotationId`、`scrollTargetAnnotation`、`pendingSelection`、`onLineSelection`、`onAddAnnotation`/`onAddAnnotationForFile`、`onAddFileComment`/`onAddFileCommentForFile`、`onEditAnnotation`、`onSelectAnnotation`、`onDeleteAnnotation`、`viewedFiles`/`onToggleViewed`、`stagedFiles`/`stagingFile`/`onStage`/`canStageFiles`/`stageError`、搜索字段、AI 字段（`aiAvailable`、`onAskAI`/`onAskAIForFile`、`isAILoading`、`onViewAIResponse`、`aiMessages`/`getAIHistoryForFile`、`onClickAIMarker`）、`prMetadata`/`prDiffScope`、`onCodeNavRequest`。Guide 区块组件可以读取与 `ReviewDiffPanel.tsx`（第 24-121 行）完全相同的字段 — 它是同一适配器的更小、专用版本，而非新子系统。

有一个字段是 dock 形态的，需要 guide 本地等价物：`focusedFilePath` / `isFocused` 推导 — 见 §6。

## 5. 每文件 patch 切片

`DiffFile`（`packages/review-editor/types.ts:8-15`）：`{ path, oldPath?, patch, additions, deletions, status }`。`parseDiffToFiles()`（`packages/review-editor/utils/diffParser.ts:31-61`）在 `^diff --git ` 边界上分割原始多文件 patch（`splitDiffChunks`，第 4-13 行），并将每个**完整块逐字**存储为 `DiffFile.patch` — 即 `patch` 已包含文件自身的 `diff --git`/`---`/`+++`/hunk 头，自包含。这正是 `getSingularPatch()` 所期望的（在 `DiffViewer.tsx:307` 和 `AllFilesCodeView.tsx:332` 中用法相同）— 由 dock 的每文件面板今天已调用 `getSingularPatch(file.patch)` 且无额外包装这一事实所证实。

**Guide 文件引用：** guide 区块可以携带文件路径列表，并通过 `ReviewDiffPanel.tsx:19-21` 已执行的精确查找解析每个路径：`state.files.find(f => f.path === path)`。无需新的索引结构 — `state.files`（一个 `DiffFile[]`）就是连接键，以 `path`（重命名后/新路径）为键。

**Hunk 级切片 — 没有现有的*可标注*先例，但存在只读先例，指向正确的方法。** `TourStopCard.tsx` 的 `AnchorBlock`（第 109-160 行）通过 `DiffHunkPreview`（`packages/review-editor/components/DiffHunkPreview.tsx`）渲染 `TourDiffAnchor`（`{ file, line, end_line, label, hunk }`），**而非** `DiffViewer`。`DiffHunkPreview` 有意为只读/紧凑：无 `lineAnnotations`、无 `ToolbarHost`、无 `renderAnnotation`、`disableLineNumbers: true`、`overflow: 'wrap'`。它处理来自 tour agent 的三种可能的 hunk 字符串形态（第 79-87 行）：完整的 `diff --git` patch、`--- `/`+++ ` 文件级 diff（前置合成的 `diff --git a/file b/file` 行）、或裸 `@@ ...@@` hunk（前置合成的 `diff --git`/`---`/`+++` 头）— 然后对结果调用 `getSingularPatch()`。这证明 `getSingularPatch` 容忍仅 hunk 输入，只要用*某种*头包装，但裸 hunk 情况下的合成 `a/file b/file` 路径是占位符，不是真实路径 — 对预览没问题，对标注导出（需要真实 `filePath`/`side`/行号以通过 `/api/feedback` 往返）则不行。

**Guide hunk 级切片的更好方法（如果 ADR 的延迟 hunk 粒度问题被解决为 hunk 级）：** 由于 guide 区块是从已解析为 `DiffFile[]` 的*同一* diff 生成的，hunk 切片应通过**从 `file.patch` 逐字**提取真实头块（`diff --git` 到 `+++`）加上选定的 `@@`-hunk 来构建 — 而非合成。这保留了真实路径和真实行号，因此结果字符串是合法的单文件（部分 hunk）patch，`DiffViewer` 可以以完整的标注保真度渲染（真实 `filePath` Prop、真实行号流入 `onAddAnnotation`/`/api/feedback`）— `DiffViewer` 不关心 patch 包含一个 hunk 还是全部，它只调用 `getSingularPatch(patch)` 并渲染其中的内容。这是一个直接的新小工具（`sliceHunksFromPatch(patch, hunkIndices)` 或类似），而非 Pierre 级变更。

**Tour 锚点"跳转到文件"现状：** `AnchorBlock` 的"Open ↗"按钮（`TourStopCard.tsx:138-143`）调用 `onAnchorClick(anchor.file)` → `TourDialog` 的处理器 → 最终 `App.tsx:367-411` 中的 `openDiffFile(filePath)`。`openDiffFile` 是 dock 原生的：它查找/创建 `REVIEW_DIFF_PANEL_ID` dockview 面板，`updateParameters({ filePath })`、`setActive()`，并设置 `activeFileIndex`。它**不会**在 Tour 对话框内联渲染 — 它在概念上关闭/后台化 tour 体验并切换 dock 到目标文件。这是现有的唯一"类 guide 界面 → 跳入真实可标注 diff"交互，其工作方式是离开类 guide 界面并重新进入 dock。该模式对 Guided Review 来说将是回归，因为 diff 在页面内联渲染（根据 ADR）— 所以 guide 区块不应复用 `openDiffFile` 的"跳到 dock"跳转；它们应直接通过 `DiffViewer` 渲染 diff。

## 6. 滚动/选择交互 — 会破坏的 dock 可见性假设

- **`focusedFilePath` 不感知 dock 可见性。** `App.tsx:1778`：`focusedFilePath: files[activeFileIndex]?.path ?? null` — 纯粹从 `activeFileIndex` 推导，没有"dock 是否实际在屏幕上"的概念。`ReviewDiffPanel.tsx:22`：`isFocusedFile = !!file && state.focusedFilePath === file.path`，作为 `isFocused` 传给 `DiffViewer`（第 77 行），`ToolbarHost`/`useAnnotationToolbar` 用它进行 §3 描述的跨实例草稿交接。**如果 guide 接管隐藏了 dock 但未触碰 `activeFileIndex`/`focusedFilePath`，隐藏的 dock 的 `ReviewDiffPanel` 仍会为该文件报告 `isFocused=true`。** 同一文件的 guide `DiffViewer` 需要自己的 `isFocused` 信号 —