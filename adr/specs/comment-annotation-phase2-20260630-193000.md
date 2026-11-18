# 规格：阶段 2 — 为 PR 评论添加标注

日期：2026-06-30
状态：草稿 → 预检

实现 ADR 004 的评论切片。设计已在讨论中锁定；本文记录该设计及预检发现的唯一修正。

## 需求（简述）

在 PR 评论时间线的每张卡片上，现有悬停操作行（View on GitHub / Copy 旁边）增加一个 **"Annotate"** 按钮。点击 → 评论框打开 → 输入备注（或 Ask AI）→ 备注显示在 Annotations 侧边栏的 "PR comments" 分组下，计入评审，并连同完整引用的评论一起发送给 agent。

仅针对评论（对整个评论的备注）。**不是**评论内的文本选择——那项延后（评论很短，仍在精简渲染器上，且选择会与卡片的点击/折叠冲突）。

## 决策（已锁定）

1. **用按钮，不用选择。** 每张卡片一个 "Annotate" 按钮。评论很短 → 整个评论即单位；按钮避免选择与卡片点击的冲突，且无需迁移渲染器。
2. **每张卡片，通过共享操作行。** 将按钮加入 `PRCommentLinkActions`（`PRCommentsTab.tsx:686`）——评论/评审卡片（`:533`）和线程卡片（`:679`）已在使用的悬停行。一处改动 → 所有卡片类型都显示。线程作为整体标注（该行位于线程第一条评论上）。
3. **标签 "Annotate"**（而非 "Comment"/"Reply"）——它从不发布到 GitHub，是给 agent 的反馈。"Comment" 读起来像回复。
4. **侧边栏：** 一个 "PR comments" 分组——作者 + 评论简短片段 + 你的备注（紧凑；完整评论可在屏幕上看到）。
5. **Agent 导出：** 作者 + **完整评论正文** + 你的备注。与代码不同（agent 可读仓库），agent 看不到 PR 评论，因此必须内联引用。
6. **Ask AI：** 复用无文件的选区范围提问，以评论正文作为 `text`，标签为 "PR comment"——与描述相同的机制。

## 预检修正（已对照代码验证）

**`exportMessageAnnotations` 是错误的导出，将不会使用。** 阅读 `parser.ts:755-812`：`MessageAnnotationEntry` 和 `exportMessageAnnotations` 是为标注 **assistant AI 消息** 而构建的——它们期望消息内基于文本锚定的 `annotations: Annotation[]`，并硬编码 "Message Feedback" / "assistant message" / "Message excerpt" 措辞。这会向 agent 错误呈现 PR 评论。ADR 004 假设其适用；实际不适用。

改为：
- **存储：** 一个简单的新类型 `CommentAnnotation { id: string; commentId: string; commentAuthor: string; commentBody: string; text: string; createdAt: number }`。无文本锚定，无块。
- **导出：** 一个小的专用格式化器 `exportCommentAnnotations(anns): string`：
  ```markdown
  # PR Comment Feedback

  ## Comment by @author
  > quoted comment body (fenced/quoted)

  your note
  ```

## 构建

### 1. 存储（App.tsx）
- `const [commentAnnotations, setCommentAnnotations] = useState<CommentAnnotation[]>([])`
- `const [selectedCommentAnnotationId, setSelectedCommentAnnotationId] = useState<string|null>(null)`
- 处理器：`onAddCommentAnnotation(commentId, author, body, note)`（构建记录，追加）、`onSelectCommentAnnotation(id)`、`onDeleteCommentAnnotation(id)`。
- `handleAskAIForComment(question, context)` → `askAI({ prompt, scope: { kind:'selection', label:'PR comment', text: context.text } })`（镜像 `handleAskAIForDescription`）。
- 通过 `ReviewStateContext`（接口 + 值 + 依赖）贯穿，与描述存储相同。

### 2. Annotate 按钮 + 弹出框（PRCommentsTab）
- `PRCommentsTab` 基于 props；它（或一个小型子组件）从 `useReviewState()` 读取添加/提问处理器——与 `AnnotatableDescription` 相同的做法。最简单：`PRCommentLinkActions` 增加 props `{ commentId, author, onAnnotate(el) }`；PRCommentsTab 拥有一个小型弹出框状态 `{ commentId, author, body, anchorEl } | null`。
- 在 `PRCommentLinkActions` 中添加 "Annotate" 按钮（View on GitHub 左侧）；onClick 以按钮元素为锚点设置弹出框状态。
- 当弹出框状态设置时，在标签页根节点渲染 `CommentPopover`（一个）：`anchorEl`、`contextText` = 截断的评论、`isGlobal: false`、`onSubmit` = 构建备注 → `onAddCommentAnnotation` → 清除状态、`onClose` = 清除、`onAskAI` = `onAskAIForComment`、`askAIContext = { kind:'selection', label:'PR comment', text: body }`。

### 3. 侧边栏 "PR comments" 分组（ReviewSidebar）
- Props `commentAnnotations?`、`selectedCommentAnnotationId?`、`onSelectCommentAnnotation?`、`onDeleteCommentAnnotation?`。
- `renderCommentAnnotationCard` 镜像 `renderDescriptionAnnotationCard`："PR comment" 标签 + `@author` + `commentBody` 片段（line-clamp）+ 备注；选择 + 删除。
- 分组渲染在 "PR description" 之后；将 `commentAnnotations.length` 计入 `totalCount`。
- 选择 → **仅在 PR Overview 面板打开时** 聚焦/滚动到目标评论卡片（与描述相同的跨面板注意事项）。v1 接受。

### 4. 计数 + 导出（App.tsx）
- `totalAnnotationCount += commentAnnotations.length`。
- `feedbackMarkdown`：非空时追加 `exportCommentAnnotations(commentAnnotations)`（新格式化器位于 `review-editor/utils/exportFeedback.ts` 或 `parser.ts`）。

## 复用映射

| 需求 | 复用 |
|---|---|
| 评论框 + Ask AI | `CommentPopover`（`onAskAI`/`askAIContext`） |
| 对选区 Ask AI | `askAI({ scope:{ kind:'selection', label, text } })` |
| 侧边栏分组 + 卡片 | 镜像 `renderDescriptionAnnotationCard` + 分组块 |
| 上下文贯穿 | 镜像描述存储字段 |
| 操作行 | `PRCommentLinkActions`（添加一个按钮） |

新增：`CommentAnnotation` 类型、`exportCommentAnnotations` 格式化器，以及每标签页的弹出框状态（无高亮器——它由按钮驱动，而非选择驱动）。

## 风险

1. **无标记持久化风险**——按钮驱动，评论 DOM 中无 web-highlighter 标记。此方面比阶段 1 更简单。
2. **跨面板聚焦**——选择侧边栏卡片仅在 Overview 面板打开时聚焦评论（与描述相同）。v1 接受。
3. **弹出框锚点稳定性**——弹出框打开时评论卡片可能折叠/滚动；锚定到按钮元素并让 `CommentPopover` 重新定位（它已在滚动时重新读取）。

## 验证

- 悬停任意卡片（评论 / 评审 / 线程）→ 操作行出现 "Annotate" → 点击 → 评论框以其为锚点打开。
- 提交 → 卡片出现在侧边栏 "PR comments" 下（作者 + 片段 + 备注）；计数 + "Send Feedback" 反映它。
- 从侧边栏删除会移除它；从框内 Ask AI 在 AI 标签页回答。
- 发送的反馈包含 "PR Comment Feedback" 部分，含 **完整引用的评论** + 备注。
- 非 PR 评审 + 描述流程不受影响。

## 预检发现（已对照代码验证，2026-06-30）

所有锚点已确认。影响接线的具体细节：

1. **`PRCommentLinkActions` 用于两处**——内联评论/评审卡片（`:533`）和 `ThreadCard` 内（`:679`）。两者都需要新 props。对于线程，标注单位是线程：传入 `commentId = thread.id`，`author`/`body` 来自 `thread.comments[0]`。
2. **`ThreadCard` 是模块级组件**（`:553`），不在 `PRCommentsTab` 闭包内——因此 `onAnnotate` 必须作为 prop 贯穿到它，再向下传到其 `PRCommentLinkActions`。（`:533` 处的内联卡片在闭包内，可直接调用 setter。）
3. **按钮样式** 镜像 View-on-GitHub 链接（`flex items-center gap-1 text-[10px] text-muted-foreground hover:text-foreground px-1.5 py-0.5 rounded hover:bg-muted/30`）。该行为 `justify-end`，所以"在其他项左侧" = 第一个 flex 子元素。
4. **通过上下文传递处理器：** `PRCommentsTab` 基于 props（`{ context, platformUser }`）但在 provider 内渲染，因此它调用 `useReviewState()` 获取 `onAddCommentAnnotation` / `onAskAIForComment`，并拥有本地 `annotating` 弹出框状态，在标签页根节点渲染一个 `CommentPopover`。与 `AnnotatableDescription` 相同的模式。
5. **`commentId` = 条目 id**（卡片上的 `data-comment-id`，`:490`/`:571`）——以便未来侧边栏选择可滚动到卡片。

**已批准。**

## 参考

- ADR：`adr/decisions/004-annotate-pr-description-and-comments-20260630-155000.md`（决策 7 中用于评论的 `exportMessageAnnotations` 在此被取代）
- 阶段 1 规格：`adr/specs/description-annotation-phase1-20260630-171500.md`