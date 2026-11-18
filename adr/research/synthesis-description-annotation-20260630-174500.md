# 综合：PR 描述标注（第一阶段）— 置信度评审

日期：2026-06-30

汇总 ADR 004 / 第一阶段规格背后的研究，并记录一次经代码验证的最终置信度检查。结论：**第一阶段的每一部分都是复用现有、已在生产运行的机制。** 有一个真实风险（标记持久化），已有已知的缓解方案。

## 我们要构建什么

在 PR 描述中选中文本 → 评论框立即打开（仅评论）→ 该注释（可选带 Ask-AI）出现在 Annotations 侧边栏的"PR description"分组下，计入评审，并在 Send Feedback 时发送给 agent。生命周期与 diff 评论相同，只是锚定在正文文本上。

## 研究轨迹

- 标注系统（会话内两次 spike）：代码评审使用 `CodeAnnotation`（按行锚定）；plan/annotate 使用 `Annotation`（通过 web-highlighter 按文本锚定）。正文文本系统最合适，且完全位于 `packages/ui`，已在第二个界面复用（iframe 上的 `useHtmlAnnotation`）。
- 渲染器：`SPIKE-renderer-migration` + `SPIKE-renderer-density-parameterization` → 已完成。描述现在通过共享的 `BlockRenderer`（`RenderedMarkdown`）渲染，带 `data-block-id`，因此 DOM 已具备标注条件。
- 本轮：对照代码验证了其余关键论断。

## 对照代码验证（置信度检查）

| 论断 | 已验证 | 证据 |
|---|---|---|
| 仅评论（选中 → 弹出框，无工具栏） | ✅ | `useAnnotationHighlighter.ts` 的 CREATE 处理器：`mode==='comment'` → `setCommentPopover(...)`；工具栏仅在 `else` 分支 |
| 恢复操作可安全重复执行 | ✅ | `applyAnnotationsInternal` 跳过已标记的 id（`getDoms`/`[data-bind-id]`），然后 `fromStore` → `findTextInDOM` 回退 |
| 点击高亮可选中它 | ✅ | `Highlighter.event.CLICK → onSelectAnnotation(id)` |
| 侧边栏已承载第二种标注类型 | ✅ | `ReviewSidebar` 的 `editorAnnotations` → `EditorAnnotationCard`，自带删除，`totalCount` 汇总两者 |
| App 持有描述正文用于导出 | ✅ | `App.tsx:302` 的 `prContext`；`feedbackMarkdown`（`:1704`）、`totalAnnotationCount`（`:1712`） |
| 对无文件选择可 Ask AI | ✅ | `AskAIParams.scope`；`buildDefaultPrompt`（`useAIChat.ts:78-82`）构造 `Re: {label}` + `Selected text:` + prompt；由 `HtmlViewer` 使用 |
| 高亮 CSS + `--focus-highlight` | ✅ | 样式位于 `packages/editor/index.css:118-161`；`--focus-highlight` 位于评审编辑器加载的主题文件中（在 plan 编辑器中可用） |

## 唯一的真实风险 — 及缓解方案

**web-highlighter 会向 React 渲染的 DOM 注入 `<mark>`；`RenderedMarkdown` 重新渲染可能将其清除。** 这是唯一需要小心的地方。

缓解方案（均为已知、低新颖性）：
1. 用 `React.memo` 包裹 `AnnotatableDescription`/`RenderedMarkdown`，使其仅在 `markdown` 或 `descriptionAnnotations` 变化时重新渲染 — 避免父组件重渲染带来的附带协调。
2. 渲染后重新应用：`useEffect(() => hook.applyAnnotations(descriptionAnnotations), [descriptionAnnotations, <markdown key>])`。幂等（已验证），因此可放心多次调用。
3. `applyAnnotations` 中的文本搜索回退：DOM 变化后若 `startMeta` 无法解析，则按 `originalText` 重新绑定。

残留问题：实时上下文 SSE 更新若在标注下方*改变描述文本*，可能丢失该标注的锚点。频率低；v1 接受。

## 置信度

**高。** 没有真正的新子系统 — 评论引擎、评论框、Ask AI、侧边栏、导出和计数都是现有机制接到新界面。新代码很少（wrapper + 一个 store + 上下文字段 + 一个侧边栏分组 + 两行导出/计数 + 移动的 CSS）。唯一的真实风险是众所周知的 React 与 DOM 注入问题，已有 plan 查看器在用的标准缓解方案。

## 可开始构建清单（来自规格）

1. `AnnotatableDescription` wrapper（`React.memo`）— 以 `comment` 模式接入 hook，仅渲染 `CommentPopover`，重新应用 effect。
2. `App.tsx` 中的 `descriptionAnnotations` store + 处理器；通过 `ReviewStateContext` 传递（值**和**依赖）。
3. `ReviewSidebar`：仿照 `editorAnnotations`/`EditorAnnotationCard` 的"PR description"分组。
4. `totalAnnotationCount` + `feedbackMarkdown` 纳入正文文本（`exportAnnotations(parseMarkdownToBlocks(prContext.body), …, 'PR Description Feedback', 'PR description')`）。
5. Ask AI：`onAskAI` → `askAI({ scope: { kind:'selection', label:'PR description', text } })`（同 `HtmlViewer`）。
6. 将 `.annotation-highlight` 样式移入 `packages/ui/theme.css`。

## 参考

- ADR：`adr/decisions/004-annotate-pr-description-and-comments-20260630-155000.md`
- 规格：`adr/specs/description-annotation-phase1-20260630-171500.md`
- 渲染器 spike：`adr/research/SPIKE-renderer-migration-20260630-155500.md`、`adr/research/SPIKE-renderer-density-parameterization-20260630-160500.md`