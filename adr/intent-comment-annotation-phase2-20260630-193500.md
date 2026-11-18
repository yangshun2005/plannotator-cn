# 意图：为 PR 评论添加标注（第二阶段）

**新增**：在 PR 评论时间线的每张卡片上添加"标注"操作，让审查者能为某条评论附加备注，备注显示在标注侧边栏，并在发送给 agent 时附带被引用的评论——与 diff 评论或描述备注相同的反馈生命周期。

**原因**：评论时间线目前是只读的。审查者希望针对某条具体评论作出回应——"这才是真正的问题"、"不同意，理由如下"——并把它并入发送给 agent 的反馈中，而无需离开审查流程。与代码不同（agent 能读取仓库），PR 评论对 agent 不可见，因此备注必须携带评论本身。

**做法**：在共享的悬停操作行（`PRCommentLinkActions`）中放置一个**"标注"**按钮，所有卡片类型都已使用该操作行——因此一处改动即可让评论、审查和讨论串都显示该按钮。点击后打开现有的共享 `CommentPopover`（含 Ask AI），锚定到该按钮；备注附加到整条评论（仅评论，不支持文本选择——评论很短，裁剪后的渲染器尚未适配标注，且选择操作会与卡片的点击/折叠冲突）。在 `App.tsx` 中新增一个简单的 `CommentAnnotation` store（`{ id, commentId, commentAuthor, commentBody, text, createdAt }`），通过 `ReviewStateContext` 传递；`PRCommentsTab` 从 `useReviewState()` 读取处理函数并持有 popover 状态，将 `onAnnotate` 回调向下传递给内联卡片和 `ThreadCard`。在侧边栏的"PR 评论"分组下展示这些备注（作者 + 评论片段 + 备注，支持选择/删除），与描述分组保持一致，并将它们计入 `totalAnnotationCount` 和侧边栏计数。将 Ask AI 接入现有的无文件范围选择询问（`label: 'PR comment'`，`text: comment body`）。通过**新的专用格式化器** `exportCommentAnnotations` 导出——作者 + 完整引用的评论文本 + 备注——追加到 `feedbackMarkdown`；刻意**不**使用 `exportMessageAnnotations`，预检表明它是为标注助手 AI 消息而构建的，会把 PR 评论错误地标记给 agent。无需更改服务端、端点或 Pi 运行时；改动仅限于 `packages/review-editor` 以及共享的 `CommentAnnotation` 类型。

---

决策记录：
- ADR：`adr/decisions/004-annotate-pr-description-and-comments-20260630-155000.md`（决策 7 中"用 `exportMessageAnnotations` 处理评论"已被取代——见规格）
- 规格（已批准，含预检发现）：`adr/specs/comment-annotation-phase2-20260630-193000.md`
- 第一阶段（参考）：`adr/specs/description-annotation-phase1-20260630-171500.md`、`adr/intent-description-annotation-phase1-20260630-180000.md`