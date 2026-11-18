# 4. 标注 PR 描述和评论，接入 agent 反馈流水线

日期：2026-06-30

## 状态

已接受

## 背景

PR Overview 面板展示 PR 描述和评论时间线，但两者都是只读的。我们希望能在描述任意位置选中并评论（类似 plan/annotate 应用），也能对单条 PR 评论评论，并让这些注释随现有 diff 标注一起进入发给 agent 的反馈。

已有两套标注系统：

- **代码审查**（`CodeAnnotation`）锚定到 diff 行（文件 + 行 + 侧）。无法锚定到普通文本。
- **Plan/annotate**（`Annotation`）锚定到渲染后 markdown 中选中的文本，使用 web-highlighter（`startMeta`/`endMeta` + `originalText`）。其引擎——`useAnnotationHighlighter` + `AnnotationToolbar` + `CommentPopover` + `FloatingQuickLabelPicker`——完全位于 `packages/ui`，不依赖 plan 特有逻辑。它已在第二个界面复用（`useHtmlAnnotation.ts`，作用于 iframe），证明其与界面无关。

PR 描述/评论通过 `PRSummaryTab.tsx` 中的 `MarkdownBody` 渲染——这是我们自定义 markdown 引擎的精简副本，只处理 5 种块类型（heading、code、list-item、blockquote、hr），对表格、HTML 块、指令、提示框没有真正的渲染。完整渲染器（`BlockRenderer` + `blocks/` 组件）使用同一个自定义解析器（`parseMarkdownToBlocks`）和完整组件集，并输出高亮器所需的 `data-block-id` DOM。

## 决策

1. **使用文本引擎，不用 `CodeAnnotation`。** 用 plan/annotate 的文本锚定 `Annotation` 系统（`useAnnotationHighlighter` + `AnnotationToolbar` + `CommentPopover` + `FloatingQuickLabelPicker`）标注 PR 描述/评论。复用该 Hook 和这三个组件——**不**复用整个 plan `Viewer`（它带有约 50 个 plan 专用 Props）。

2. **升级渲染器。** 用完整的共享块渲染器替换 PR 文本的精简 `MarkdownBody`。将块分发逻辑从 `Viewer` 中抽出，做成小型共享组件 `RenderedMarkdown`，让 plan 查看器和 PR 面板共用同一个渲染器（不重复）。这同时修复 HTML/表格/提示框渲染，并使文本可标注。

3. **两种交互方式——仅评论（有意比 plan/annotate 更简单）。**
   - **描述** → 选中文本后**立即打开评论框**。仅评论：无工具栏、无快捷标签、无删除/红线选择器。对应高亮器的 `comment` 模式（选中 → 直接 `CommentPopover`）。这是有意区别于 plan/annotate（后者显示多选项工具栏）。
   - **每条评论** → 卡片上的小"评论"按钮，打开同一个 `CommentPopover`，给该评论附加注释。评论卡片内不支持选中文本（避免与卡片的点击/折叠处理冲突）。

4. **一个评论弹窗，带 Ask AI。** 使用现有共享 `CommentPopover`——代码审查中用于文件评论的同一个组件。接通其 **Ask AI** 动作，以便对选中的描述文本询问 AI（复用审查编辑器现有的 Ask AI 主干）。不新增弹窗。

5. **独立 store。** 新增一个 `Annotation[]`（文本）store，提升到 `review-editor/App.tsx`，通过 `ReviewStateContext` 传递，与现有 `CodeAnnotation[]` 区分。两个模型在内存中保持独立。

6. **在 Annotations 侧边栏的"PR description"分组下展示。** 描述评论渲染在现有审查 Annotations 侧边栏中，位于独立的"PR description"分组，与 diff 评论并列——可在其中选中、编辑、删除，与任何标注一样。侧边栏已渲染第二种标注类型（`editorAnnotations`，有自己的删除路径），因此沿用该既有模式，而非新造合并逻辑。删除/选中因此由侧边栏"免费"提供——高亮上无需单独操作入口。

7. **接入现有流水线，不改服务端。** 文本标注计入 `totalAnnotationCount`（使"Send Feedback"出现），并通过 `exportAnnotations`（描述）/ `exportMessageAnnotations`（评论）追加到 `feedbackMarkdown`。走现有 `/api/feedback` POST——无新端点。

8. **分阶段。** 先做描述（端到端验证完整流水线——选中 → 评论 → 侧边栏 → 发送），再做评论。

## 影响

- 审查编辑器将同时持有两个标注模型（diff 用 `CodeAnnotation[]`，文本用 `Annotation[]`）。它们在内存中保持独立，仅在展示边缘合并——Annotations 侧边栏（新增"PR description"分组，沿用现有 `editorAnnotations` 的渲染/删除模式）和导出的反馈 markdown。
- 将描述渲染器换成 `RenderedMarkdown` 会改变面板的间距/样式，需要重新对齐。
- 文本标注没有真实的文件/行号；导出时按来源标注（"PR description" / "comment by @x"）而非 `file:line`。`exportAnnotations` 在无行号时已能优雅降级。
- 抽取 `RenderedMarkdown` 会触及 plan `Viewer`（它开始使用共享组件）。风险低，但属于共享代码改动，并非仅限审查编辑器。
- 无服务端、端点或 Pi 运行时改动——工作限于 `packages/ui`（共享渲染器 + 复用的标注组件）和 `packages/review-editor`（状态、接线、每条评论的按钮）。
- 评论卡片点击冲突已在设计上规避（用按钮而非选中），因此不构成运行时风险。