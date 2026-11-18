# SPIKE：将 PR 描述/评论渲染器迁移到完整块渲染器

日期：2026-06-30

## 问题

在给 PR 描述/评论添加标注之前，我们希望获得与 plan/annotate 应用相同的完整 Markdown 渲染。当前渲染器究竟缺什么？"完整"渲染器由什么构成？它有多可复用？切换的风险是什么？

## 现状：两个渲染器，一个解析器

两个渲染器都用**同一个**自定义引擎解析——`parseMarkdownToBlocks`（`packages/ui/utils/parser.ts`）。区别仅在于对块的处理方式。

- **精简版（`MarkdownBody`）**——`packages/review-editor/components/PRSummaryTab.tsx`。本地 `switch (block.type)` 只处理 **heading、code、list-item、blockquote、hr**，外加默认段落路径。行内内容使用 review-editor 自己的 `renderInlineMarkdown`（`utils/renderInlineMarkdown.tsx`）；原始 HTML 经 DOMPurify 行内净化（`SafeHtml`）。
  - **缺失：** `table`、`html`、`directive` 和 GitHub `alert` 块的专门渲染。它们都落入段落路径，因此表格/标注/HTML 块渲染为纯文本或行内净化文本——而非表格/标注。
  - **用于两处：** PR **描述**（PRSummaryTab）和 `PRCommentsTab.tsx` 中的**每条评论/审查/线程/回复正文**。两者都传入 `textClassName`（最近添加）来控制基础文本大小。

- **完整版（`BlockRenderer`）**——`packages/ui/components/BlockRenderer.tsx`。`switch (block.type)` 覆盖 **heading、blockquote（+ alert）、list-item、code、table、hr、html、directive、paragraph**。它委托给共享块组件（`blocks/CodeBlock`、`TableBlock`、`HtmlBlock`、`Callout`、`AlertBlock`，以及 `InlineMarkdown`、`ListItemBody`）。每个块都带 `data-block-id` 渲染。

## "完整渲染器"究竟是什么

有两层，重量差异很大：

1. **`BlockRenderer`——干净、可复用的核心。** 所有 props 都是可选回调（`onOpenLinkedDoc`、`onOpenCodeFile`、`imageBaseDir`、`onImageClick`、`onToggleCheckbox`、`githubRepo`、`onNavigateAnchor`、`orderedIndex`）。**不依赖 plan/session/store。** 它自己分发 `code → CodeBlock` 和 `table → TableBlock`，因此单独就能正确渲染所有块类型。它引入的块组件（`CodeBlock` = hljs，`HtmlBlock` = marked + DOMPurify，`TableBlock` = 复制工具栏/弹出，`Callout`/`AlertBlock` = `renderProseBody`）都在 `packages/ui` 中，接收 `block` + 可选回调。干净。

2. **Viewer 渲染循环——沉重、plan 专用。** `Viewer.tsx:641-720` 用一堆 plan 专用行为包裹 `BlockRenderer`：`groupBlocks`（有序列表索引的列表分组）、`MermaidBlock`/`GraphvizBlock`（图表渲染）、代码块和表格**悬停工具栏**（plan 模式下用于代码块标注）、灯箱、frontmatter 卡片，以及 `pinpoint` 输入法。这个循环**不是** PR 面板想要的。

**结论：** 真正可复用的单元是 `BlockRenderer`（+ `blocks/` 组件），而非 Viewer 的循环。ADR 004 说"把块分发从 Viewer 中提取到共享的 `RenderedMarkdown`"。准确的说法更窄：在 `BlockRenderer` 之上构建一个**精简**的 `RenderedMarkdown`（`groupBlocks` + map → `BlockRenderer`，带有序列表索引，**不含**图表/工具栏/灯箱），并在 PR 面板中复用它。我们**不需要**重构 Viewer 的丰富循环来共享一个渲染器——那会把所有 plan 关注点拖进共享组件，是要避免的陷阱。`BlockRenderer` 已经是两条路径共同依赖的共享单元。

## 真正的风险：样式/密度

`BlockRenderer` 的类是为**整页 plan 文档**硬编码的：`h1 = text-2xl mb-4 mt-6`、`h2 = text-xl mt-8`、段落 `text-[15px] mb-4`、`hr my-8` 等。PR Overview 面板是**紧凑**的（12–13px 文本，间距紧），评论时间线更密。直接放入 `BlockRenderer` 会让描述读起来像整页 plan 文档——标题过大、垂直节奏过重——却在小面板里。

所以迁移不只是"换组件"。它需要**大小/密度控制**：

- 当前 `MarkdownBody` 已为此暴露 `textClassName`。替代品必须保留等效控制。
- 选项（在 spec 中决定）：给 `BlockRenderer`/`RenderedMarkdown` 参数化密度/大小变体；或用覆盖大小的 prose-scope 类包裹输出；或向下传大小 token。块组件的硬编码类（`BlockRenderer` 和 `renderProseBody` 中的 `text-[15px]`）是需要灵活的部分。

这个样式协调——对描述**和**评论正文都是——是真正工作的大头，也是最容易出错的地方。

## 次要考虑

- **图表。** PR 描述可能包含 ` ```mermaid ` 块。精简渲染器（不含 Viewer 的 Mermaid/Graphviz 分发）将它们渲染为代码块，而非图表。v1 可接受；接入 `MermaidBlock`/`GraphvizBlock` 是增量后续。
- **行内渲染器替换。** 迁移到 `BlockRenderer` 会把 review-editor 的 `renderInlineMarkdown` 换成共享的 `InlineMarkdown`（额外处理图片、链接文档/代码文件链接、锚点）。很可能是升级；需视觉检查行内样式是否匹配紧凑面板。
- **标注收益。** `BlockRenderer` 在每个块上输出 `data-block-id`——正是标注 Hook 想要的 DOM。所以渲染器迁移也是让正文可标注的前提；两者设计上耦合，这就是我们先做这个的原因。
- **共享代码影响范围。** 复用 `BlockRenderer` 风险低（已通过 plan viewer 在生产中）。构建新的精简 `RenderedMarkdown` 是增量的。v1 可完全不动 `Viewer`。

## 待 spec 解决的开放问题

1. **密度机制：** 给 `BlockRenderer`/`RenderedMarkdown` 参数化大小变体，还是用覆盖的 prose-scope 类包裹？（影响描述和评论。）
2. **一个渲染器还是两个消费者：** 新精简 `RenderedMarkdown` 仅用于 PR 面板（不动 Viewer），还是之后也把 Viewer 重构到它上面？建议：v1 仅 PR 面板，不动 Viewer。
3. **评论：** 评论正文也迁移到完整渲染器（一致性 + 更好的 HTML/表格），还是 v1 只做描述？评论正文是更密的表面，最考验密度问题。
4. **图表：** v1 把 mermaid/graphviz 渲染为代码（简单），还是现在就接入图表块？

## 涉及文件

- `packages/review-editor/components/PRSummaryTab.tsx`（`MarkdownBody`，精简渲染器）
- `packages/review-editor/components/PRCommentsTab.tsx`（所有评论正文使用 `MarkdownBody`）
- `packages/review-editor/utils/renderInlineMarkdown.tsx`（review-editor 行内渲染器）
- `packages/ui/components/BlockRenderer.tsx`（完整、可复用的核心）
- `packages/ui/components/Viewer.tsx:641-720`（沉重的 plan 渲染循环）、`:953`（`groupBlocks`）
- `packages/ui/components/blocks/`（`CodeBlock`、`TableBlock`、`HtmlBlock`、`Callout`、`AlertBlock`、`proseBody`）
- `packages/ui/components/InlineMarkdown.tsx`、`ListItemBody.tsx`
- `packages/ui/utils/parser.ts`（`parseMarkdownToBlocks`，两个渲染器共享）
```