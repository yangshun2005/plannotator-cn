# 规格：阶段 1 — 在 PR 描述上评论

日期：2026-06-30（第 3 版 — 最终版，已对照代码验证）
状态：最终版 — 每一部分都已对照代码验证为复用（见 `adr/research/synthesis-description-annotation-20260630-174500.md`）。可以开始构建。

实现 ADR 004 的描述部分。渲染器前置条件已完成：描述通过 `RenderedMarkdown` 渲染，每个块都带 `data-block-id`，因此 DOM 已具备标注条件。

## 需求（简述）

在 PR 描述中选中文本 → **立即弹出评论框** → 输入评论（或就此询问 AI）→ 评论出现在 **Annotations 侧边栏的 "PR description" 分组下**，与 diff 评论完全一致：可在那里选中/编辑/删除，计入审查，并在 Send Feedback 时发送给 agent。

仅评论。无工具栏、无快捷标签、无删除/红线选择器。刻意比 plan/annotate 更简单。

## 流程

`在描述中选中文本`
→ web-highlighter 触发 → Hook 处于 **`comment` 模式** → 直接打开 `CommentPopover`（无工具栏）
→ 用户输入评论（或点击 **Ask AI**）→ 提交
→ 构建 `Annotation`（`originalText` + `startMeta`/`endMeta`），高亮文本，加入 `descriptionAnnotations` store
→ 卡片出现在 Annotations 侧边栏的 **"PR description"** 下；`totalAnnotationCount` 递增 → 显示 "Send Feedback"
→ Send Feedback → `feedbackMarkdown` 包含 "PR Description Feedback" 小节 → 现有 `/api/feedback` POST

## 构建

### 1. 描述上的标注引擎 — 仅评论
描述是直接的 DOM 容器，因此**直接**挂载 `useAnnotationHighlighter`（无 iframe）。在新的 `AnnotatableDescription` 包装组件中（PRSummaryTab 用它替代 `RenderedMarkdown` 渲染；PRSummaryTab 保持基于 props，包装组件从 `useReviewState` 获取 store）：
- `containerRef` **仅**包裹描述的 `RenderedMarkdown`（不含 `ChecksDisclosure`）。
- `useAnnotationHighlighter({ containerRef, annotations: descriptionAnnotations, onAddAnnotation, onSelectAnnotation, selectedAnnotationId, mode: 'comment' })`。
- **仅**渲染 `<CommentPopover>`（由 `hook.commentPopover` + `hook.handleCommentSubmit`/`handleCommentClose` 驱动）。**不要**渲染 `AnnotationToolbar` 或 `FloatingQuickLabelPicker` — comment 模式跳过工具栏。
- 变更时重新应用：`useEffect(() => hook.applyAnnotations(descriptionAnnotations), [descriptionAnnotations, <content key>])`，使标记在重新渲染后保留（见风险）。

### 2. Store（App.tsx）
- `const [descriptionAnnotations, setDescriptionAnnotations] = useState<Annotation[]>([])`
- `const [selectedDescriptionAnnotationId, setSelectedDescriptionAnnotationId] = useState<string|null>(null)`
- 处理器：`onAddDescriptionAnnotation(ann)`（追加 + 选中）、`onSelectDescriptionAnnotation(id)`、`onDeleteDescriptionAnnotation(id)`。
- 与 plan 编辑器的精简接口一致（`packages/editor/App.tsx:2857-2868, 2912`）。

### 3. 贯穿 `ReviewStateContext`
将 store + 处理器加入 `ReviewState` 接口、provider 值对象**及其 deps 数组**（它是一个大的 `useMemo` — 漏掉依赖 = 数据陈旧）。导入 `Annotation` 类型。`AnnotatableDescription` 通过 `useReviewState()` 读取它们。

### 4. 侧边栏 — "PR description" 分组
`ReviewSidebar` 已渲染第二种标注类型（`editorAnnotations`，带 `onDeleteEditorAnnotation`）— **照搬该模式**：
- 添加 props `descriptionAnnotations?: Annotation[]`、`onSelectDescriptionAnnotation`、`onDeleteDescriptionAnnotation`。
- 渲染 **"PR description"** 卡片分组（作者、评论文本、引用的 `originalText`）。并入 `totalCount`。
- 点击卡片 → `onSelectDescriptionAnnotation(id)` → 设置 `selectedDescriptionAnnotationId` → Hook 滚动到描述中的高亮并添加 `.focused`。
- 删除卡片 → `onDeleteDescriptionAnnotation(id)` → `hook.removeHighlight(id)` + 从 store 移除。
- 跨面板说明：选中描述卡片时，仅当 **PR Overview 面板已打开**才聚焦高亮。若已关闭，要么先打开它，要么不操作。（v1：打开/聚焦 overview 面板 — 改动小，或接受不操作。）

### 5. 计数 + 导出（App.tsx）
- `totalAnnotationCount`（`:1712`）`+= descriptionAnnotations.length`。
- `feedbackMarkdown`（`:1704`）追加 `exportAnnotations(parseMarkdownToBlocks(prContext.body), descriptionAnnotations, [], 'PR Description Feedback', 'PR description')`。`App` 已持有 `prContext`（`:302`）。无需服务端改动。

### 6. 高亮 CSS
`.annotation-highlight` + `.deletion`/`.comment`/`.focused`/`:hover` 仅存在于 `packages/editor/index.css:118-161`。移入 `packages/ui/theme.css`（共享；约 40 行），使两个编辑器都能使用。`--focus-highlight` 已在 review 编辑器加载的主题文件中定义（在 plan 编辑器中可用），因此 `.focused` 是安全的。

### 7. 从评论框 Ask AI — 复用（已支持）
并非新工作。`AskAIParams` 有一等公民 **`scope`** 字段，`buildDefaultPrompt`（`useAIChat.ts:78-82`）已构建**带标签、无文件的 "selection" 询问** — `Re: {label}` + `Source:` + `Selected text: …` + 问题。它存储在 `AIQuestion` 上，正是 HTML 查看器通过 `CommentPopover` 的 `askAIContext` 传入的内容。因此：
- 在 App 中添加 `handleAskAIForDescription(question)`：`askAI({ prompt: question, scope: { kind: 'selection', label: 'PR description', text: selectedText } })` — **无 filePath**。
- 为描述 `CommentPopover` 接线 `onAskAI={handleAskAIForDescription}` 和 `askAIContext={{ kind: 'selection', label: 'PR description', text: selectedText }}`，照搬 `HtmlViewer.tsx:313-325`。
- 答案落在 AI 侧边栏。由于 `scope` 存储在问题上，卡片携带其 "PR description" 上下文。目前无文件询问归入 "general"（`AITab.tsx:75`）；可选的一分支调整按 `question.scope?.label` 分组，使 PR 描述询问归入各自标题下。锦上添花，非必需。

无新 AI 管道 — 与 HTML 查看器已使用的机制相同。

## 复用映射

| 需求 | 复用（无改动） |
|---|---|
| 选中 → 评论框 | `useAnnotationHighlighter` 的 `mode: 'comment'` |
| 评论输入 + Ask AI | `CommentPopover`（`onAskAI`/`askAIContext`） |
| 消费方模板 | `HtmlViewer.tsx:295-328`（仅 `CommentPopover` portal） |
| store 形状 | `editor/App.tsx:2857-2868, 2912` |
| 侧边栏第二类型模式 | `ReviewSidebar` 的 `editorAnnotations` 路径 |
| 对选区 Ask AI | `askAI({ scope: { kind:'selection', label, text } })`（`useAIChat.ts:78-82`）— 同 `HtmlViewer` |
| 导出 | `exportAnnotations(blocks, anns, [], title, subject)` |

## 风险

1. **React 与 web-highlighter 标记（主要）。** Hook 向 React DOM 注入 `<mark>`；`RenderedMarkdown` 重新渲染可能覆盖它们。缓解措施（均已验证/标准）：(a) 用 `React.memo` 包裹 `AnnotatableDescription`，使其仅在 `markdown`/`descriptionAnnotations` 变化时重新渲染 — 避免附带协调；(b) 在 `useEffect` 中重新运行 `applyAnnotations`，依赖 `[descriptionAnnotations, <markdown key>]` — `applyAnnotationsInternal` 是**幂等**的（跳过已标记的 id），因此可放心频繁调用；(c) 若 `startMeta` 不再解析，回退到 `findTextInDOM(originalText)`。验证标记在面板重新渲染和 PR 上下文 SSE 推送后仍保留。
2. **实时上下文刷新。** 描述正文可能通过 SSE 变化；标注的文本锚点可能无法重新绑定。频率低；v1 接受（回退到文本搜索或丢弃）。

既然 Ask AI 已确认为复用（§7），阶段 1 全部为复用 — 无真正的新子系统。风险 1（标记持久性）是唯一真正的风险。

## 已锁定决策

- 仅评论，`comment` 模式，仅 `CommentPopover`。✓
- 显示在 Annotations 侧边栏的 "PR description" 下；在那里选中/删除。✓
- Ask AI = 复用现有 `scope`-selection 询问（同 HtmlViewer）；答案在 AI 侧边栏，带其 "PR description" 上下文。可选：按 `scope.label` 分组 AI 标签页。✓
- 描述优先；评论（每卡片按钮）为阶段 2。

## 验证

- 选中描述文本 → 评论框立即打开（无工具栏）→ 添加评论 → 高亮保留。
- 卡片出现在 Annotations 侧边栏的 "PR description" 下；计数包含它；出现 "Send Feedback"。
- 选中卡片 → 滚动/聚焦高亮；删除卡片 → 高亮 + 条目消失。
- 从评论框 Ask AI → 答案显示在 AI 侧边栏。
- 发送的反馈包含 "PR Description Feedback" 小节。
- 标记在面板重新渲染 + PR 上下文刷新后保留。Plan 编辑器 + 非 PR 审查不受影响。

## 预检发现（已对照代码验证，2026-06-30）

所有规格锚点已确认。浮现三处小的实现补充 — 均不改变设计：

1. **新侧边栏卡片。** `EditorAnnotationCard` 类型为 `EditorAnnotation`（label + filePath），不适合散文式 `Annotation` 形状。构建一个小的 `DescriptionAnnotationCard`（评论文本 + 引用的 `originalText` + 作者 + 删除 + `isSelected`），在 "PR description" 分组中渲染，位置正是 `editorAnnotations` 块所在处（`ReviewSidebar.tsx:363-382`）。
2. **两个计数器，而非一个。** 更新 **`App.tsx:1712`** 的 `totalAnnotationCount`（控制 Send Feedback）**和** `ReviewSidebar.tsx:149` 的 `totalCount`（侧边栏计数/空状态），均包含 `descriptionAnnotations.length`。
3. **导出细节。** `exportAnnotations` 以 `# {title}` 开头，按 `blockId`/`startOffset` 排序 — 散文标注两者都有，且块 id 匹配，因为我们解析的是同一个 `prContext.body`。追加到 `feedbackMarkdown` 时，前置 `\n\n` 并对 `prContext?.body` 做保护。

已确认与规格完全一致：`exportAnnotations` 签名（`blocks, annotations, [], title, subject`）、`CommentPopover` props（`anchorEl/contextText/isGlobal/initialText/onSubmit/onClose/onAskAI/askAIContext`）、`feedbackMarkdown`/`totalAnnotationCount` 插入点、`ReviewStateContext` 的 `// Annotations` 槽位，以及侧边栏调用 props。

**已批准。**

## 参考

- ADR：`adr/decisions/004-annotate-pr-description-and-comments-20260630-155000.md`
- 渲染器探索：`adr/research/SPIKE-renderer-migration-20260630-155500.md`、`adr/research/SPIKE-renderer-density-parameterization-20260630-160500.md`