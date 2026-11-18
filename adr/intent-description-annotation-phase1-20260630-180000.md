# 意图：对 PR 描述添加评论（第一阶段）

**新增**功能：可在 PR 描述中选择文本并留下评论，其行为与其他评审评论一致——显示在 Annotations 侧边栏、计入评审，并在 Send Feedback 时发送给 agent。

**原因。** PR Overview 面板以只读方式渲染描述。评审者希望标记其中特定部分——"这个说法有误"、"这里需要澄清"——并让这些反馈与 diff 评论一起送达 agent，无需退出评审或重新输入上下文。目前无法在代码评审中标注正文。

**做法。** 在描述上复用现有的正文标注引擎（`useAnnotationHighlighter` + `CommentPopover`），该描述已通过共享的 `RenderedMarkdown` 渲染（因此其 DOM 带有 `data-block-id`，可直接用于标注）。以**仅评论**模式运行（`comment` 模式 → 选中文本即直接打开评论框；无工具栏、快速标签或红线标注），刻意比 plan/annotate 更简单。将描述包裹在记忆化的 `AnnotatableDescription` 中，由它挂载该 Hook、仅渲染 `CommentPopover`，并在变更时重新应用高亮（幂等，因此安全）。在 `review-editor/App.tsx` 中新增 `descriptionAnnotations: Annotation[]` 存储，通过 `ReviewStateContext` 传递，与 diff 的 `CodeAnnotation[]` 分开。在现有 Annotations 侧边栏中以"PR description"分组展示这些评论，通过一个小的新 `DescriptionAnnotationCard` 实现，沿用既有的 `editorAnnotations` 渲染/删除模式（因此选择和删除来自侧边栏——高亮上无额外操作入口）。将评论框的 Ask AI 接到现有的无文件 `scope` 选择询问（`askAI({ scope: { kind:'selection', label:'PR description', text } })`），与 HTML 查看器完全一致。将正文计数同时纳入应用级 `totalAnnotationCount` 和侧边栏的 `totalCount`，并通过 `exportAnnotations(parseMarkdownToBlocks(prContext.body), …, 'PR Description Feedback', 'PR description')` 将备注追加到 `feedbackMarkdown`。将 `.annotation-highlight` 样式移入共享的 `packages/ui/theme.css`。无服务端、端点或 Pi-runtime 变更；工作范围限于 `packages/ui`（移动的 CSS）和 `packages/review-editor`（包装器、存储、上下文、侧边栏分组、计数/导出行）。

---

研究与决策记录：
- ADR：`adr/decisions/004-annotate-pr-description-and-comments-20260630-155000.md`
- 规格（最终，已批准）：`adr/specs/description-annotation-phase1-20260630-171500.md`
- 综合 / 置信度：`adr/research/synthesis-description-annotation-20260630-174500.md`
- 渲染器研究：`adr/research/SPIKE-renderer-migration-20260630-155500.md`、`adr/research/SPIKE-renderer-density-parameterization-20260630-160500.md`