# Plannotator Inbox：线程状态模型

日期：2026-10-12。本文描述线程在 Inbox 列表中的位置及其移动原因。本文依据代码编写，代码也据此校验：`packages/shared/inbox/store.test.ts`（"线程状态模型"）将生成的和手写的事件序列分别跑在真实 store 和一份独立实现的规则模型上，并在每一步后检查所有不变量。手机应用回放 `packages/shared/inbox/fixtures/thread-state/v1/` 中的 trace fixture（见"Trace fixture"）。

规则所在位置：store（`packages/shared/inbox/store.ts`：`listRow`、`readAt`、`markSeen`、`markRead`、`markUnread`、`tellSetAside`、`markAgentChecked`、`resolveThread`、`sendMessage`、`sendReply`、`savePicks`）、分区（`packages/shared/inbox/list.ts`）、路由（`packages/server/inbox.ts`）、计数（`packages/inbox/held.ts` 的 `waitingCount`）以及线上协议（`adr/implementation/inbox-mobile.md` 第 2 节）。

## 术语

- **线程（Thread）。** 列表中的一行。其第一条消息是**根消息（root）**；根消息的记录承载线程的外观、已读、归档和 agent 已读字段。
- **Agent 消息。** agent 发送的消息（`send_message`、`submit_guide`）。它可以提问（`:::question` 块）。
- **Person 消息。** person 的 Send（一次回复），或 Inbox 自己的**备注（note）**：Mark read 发给提问 agent 的搁置备注，以及迟到的 Undo 发出的重新打开备注。备注以 person 消息的形式写入，因此像普通回复一样到达 agent（唤醒、`read_thread`），但它们不是 person 的回复，也不是一次查看（`@plannotator/core/inbox-types` 中的 `isInboxNote`）。列表派生的一切都跳过它们读取：查看、`unseen`、`sent`（规则 7）、"came back"、`waiting_on_person`，以及窗口中的查看边界（`packages/inbox/folding.ts`）、"Saved for / Delivered to" 行和 held 列表的 agent 活动（`last_agent_at`）。
- **Seq。** Inbox 全局的一个计数器。每一行都有一个；下文的"之后"和"之前"均指 seq 顺序。
- **查看（The look）。** person 查看了线程（`POST /api/inbox/threads/:id/seen`）：窗口在线程打开时记录一次，当线程打开期间到达的 agent 消息在屏幕上被看到时再记录一次（即 seen on sight，`packages/inbox/seenOnSight.ts`）。根消息的 `person_seen_seq` 变为最新 agent 消息的 seq；若给出可选的 `through_seq`（窗口展示的最新消息），则变为不高于它的最新 agent 消息的 seq：窗口读取线程之后才到达的消息仍算新消息。查看永不回退（上限等于或早于已记录的查看时不写入），上限超过最新消息时取最新消息。person 自己的回复算作对其之前所有内容的查看。Inbox 的备注不是 person 的回复，也不是一次查看。
- **Mark read。** `POST /api/inbox/threads/:id/read`。它在根消息上写入 `person_read_seq` 和 `person_read_at`，算作一次查看，并将所有仍在等待 person 的问题（open，或已 picked 但未发送）以 `set_aside` 的跳过方式关闭，`set_aside` 指明该次 Mark read（`set_aside.read_seq`，即其 `person_read_seq`）。只要其后除它自己的搁置备注外没有其他消息，它就保持有效。
- **Undo。** `POST /api/inbox/threads/:id/unread`。它清除已读字段，恢复 Mark read 计入的查看，并重新打开 Mark read 关闭的问题（根消息当前的 `person_read_seq`），恢复原状（包括已保存的 pick）。更早的 Mark read 关闭的问题保持关闭（owner 裁定 2026-10-12）。早于 `read_seq` 的记录按时间匹配（`set_aside.at` 等于根消息的 `person_read_at`）。根消息上没有 Mark read 时（Undo 之后，或 Unarchive 丢弃了它），Undo 不写入任何内容。
- **Archive。** Resolve（`POST /api/inbox/messages/:id/resolve`，`resolved: true`）。设置根消息的 `resolved_at`。**归档线（archive line）**是设置当前 `resolved_at` 的根消息行的 seq；它由日志派生，从不写入。归档线之前的消息**被归档关闭**：其问题读作 `closed`，对它们的 pick 和 Send 被拒绝（`409 thread_resolved`）。
- **Came back。** agent 在归档线之后写入了已归档线程（`agent_after_archive`），且其中仍有内容在等待某人（`back_from_archive`）：归档线之后 person 尚未查看的 agent 消息、处于 open 或 picked 的活跃问题，或归档线之后 person 的真实回复且尚无 agent 读取（备注不计），除非其后仍有效的 Mark read。在最新的归档后 agent 消息之后所做的 Mark read 会结束 "came back"：线程立即重新变为 Archived，即使 person 在那里的回复仍在等待 agent。线程保持 resolved。
- **Agent read。** `read_thread` 读取了线程。根消息的 `agent_checked_seq` 变为它所覆盖的最新 person 消息的 seq。
- **Person 发起的线程。** Compose（`POST /api/inbox/messages`）：根消息是 person 的消息（`started_by_person`），发给一个活跃会话（`to`）。路由只索引 agent 根消息，因此只有 `reply_to` 回复会落入其中（或，一旦 agent 命名了它，使用该名称的发送）。其目标会话的**投递（delivery）**（连接将其作为一轮放入会话）算作截至该点的 agent 读取。
- **未发送的工作（Unsent work）**（行的 `unsent`，批准 "inbox-reply-drafts"，2026-10-20）。person 在线程中尚未发送的内容：在其回复框中键入的文字（**回复草稿**，`inbox/reply-drafts/<thread id>.json`，位于事件日志之外，因此没有行、没有 seq，agent 也看不到）、已保存但未发送的 pick，以及等待 Send 的评论（关于附件、引导式审查、消息）。在 person 键入时保存（对 `updated_at` 做 compare-and-set）；Discard 和携带这些文字的 Send（`clear_draft`，指明保存时的 `updated_at`，或一个恰好包含该回复文字的草稿，以便来自另一窗口的更新、不同的草稿得以保留）会删除草稿；被删除的线程连同其草稿一起删除。
- **等待 person（Waiting on the person）**（摘要的 `waiting_on_person`，Workspaces 的线上协议）。线程未 resolved，其最后一条消息（跳过 Inbox 的备注）是 agent 的，且没有有效的 Mark read：有效的 Mark read 意味着没有内容在等待 person。模型在每一步后检查它。

## 问题状态

派生得出，从不存储，按此顺序：

| 状态 | 条件 |
|---|---|
| `superseded` | 线程中较晚的 agent 消息替换了它（`superseded_by`） |
| `closed` | 其消息被归档关闭，或 Mark read 将其搁置（`set_aside`） |
| `open` | 无答案 |
| `sent` | 答案随 Send 发出（`sent_revision` 等于 `revision`）。由执行跳过的 Send 所做的跳过算作已发送的答案 |
| `picked` | 答案已保存但未发送 |

## 分区

六个分区，按此固定顺序显示：Stopped on you、Holding up work、Waiting on you、New since you looked、Sent、Quiet。线程恰好属于其中一个。第一条成立的规则胜出：

| # | 规则 | 分区 |
|---|---|---|
| 1 | 已归档：resolved 且未 came back | Quiet |
| 2 | Mark read 有效（线程 resolved 时，在归档线之后做出） | Quiet（"Read"）；当有未发送的工作等待且没有 Re-ask 备注等待时，或当 Mark unread 收回了查看（`unseen` 大于 0）时为 New |
| 2a | 每个 open 问题都列在一个仍在等待的 Re-ask 备注中（批准 "inbox-reask"）。备注一直等待，直到它发往的会话（其 `to`，否则为提问者）再次写入，且仅当它列出的某个问题仍为 open | Sent |
| 3 | 一个 open 问题带有 `Stopped:` | Stopped on you |
| 4 | 一个 open 问题带有 `Holds up:` | Holding up work |
| 5 | 任何其他 open 问题 | Waiting on you |
| 6 | 已保存但未发送的 pick，无 open 问题 | New since you looked（"Answered, not sent"） |
| 7 | 最后一条消息是 person 的，且无 agent 读取 | Sent |
| 8 | 查看之后的 agent 消息 | New since you looked |
| 9 | 未发送的工作：回复草稿、pick、等待的评论 | New since you looked（"Draft · …"） |
| 10 | 其他任何情况 | Quiet |

规则 9（以及规则 2 中的例外）是该批准项的规则：持有未发送工作的线程永不沉入 Quiet。归入 New，是因为 New 已包含规则 6 的 "Answered, not sent"（仅等待 person 的 Send 的工作），且位于 Quiet 之上而不声称有问题在等待。仅因 person 自己的文字或评论而位于 New 的行（无未查看的 agent 消息、无规则 6）不加粗，也不计入等待 person；Drafts 行改为计入它。规则 1 优先于它：已归档线程保持归档并保留其草稿，因为归档是 person 明确的"完成"（归档已关闭问题，且无法向其添加评论）。

在分区内，按线程最新消息（`last_at`）从新到旧排列，相同则按线程 id。仅当线程获得消息时，行才在其分区内移动；查看、pick、已读、回复草稿和归档会在分区之间移动它。

**Archived** 是第七个位置（六个分区加 Archived）：resolved 且未 came back。came-back 线程像 open 线程一样由规则 2 至 10 放置。

## 两种列表视图

| 视图 | 路由 | 已归档线程 |
|---|---|---|
| 默认 | `GET /api/inbox/threads`（`archived=include`） | 在 Quiet 中，像任何 quiet 线程一样。手机的门原样转发此路由，因此绘制分区的手机保留每个线程 |
| 窗口的 | `GET /api/inbox/threads?archived=exclude` 加上 `GET /api/inbox/threads/archived` | 不在任何分区中；由归档路由按最新归档优先列出（`resolved_at`，相同则按线程 id），Settings 的 Archived 块读取它 |

came-back 线程在两种视图中都位于分区中，且不在归档列表中。

## 事件

| 事件 | 路由（窗口） | 写入内容 |
|---|---|---|
| Agent 消息，新线程 | `send_message` 不带 `reply_to` | 一个根消息。路由：带 `thread` 名称时，项目中该名称的最新 open 线程，否则新建线程。不带名称时，总是新建线程（owner，2026-11-08：加入会话最后一个未命名线程会把长寿命会话的新闻放入数周前无关的线程）。已归档线程永不加入 |
| Agent 回复 | `send_message` 带 `reply_to` | 该线程中的一条消息，无论是否归档。已归档：它落在归档线之后 |
| Agent 问题 | 上述任一，带 `:::question` 块 | 该消息，以及每个块一条问题记录 |
| Supersede | `supersedes: [...]` 或 `Replaces:` 行 | 该消息，然后对每个被命名且为 open 或 picked 的问题写入 `superseded_by`。命名其他任何内容会拒绝整个发送（`invalid_supersedes`） |
| Open（seen） | `POST /api/inbox/threads/:id/seen`，可选 `through_seq` | 当有 agent 消息（不高于 `through_seq`，若给出）未查看时写入 `person_seen_seq`；否则不写入 |
| Pick | `POST /api/inbox/messages/:id/picks` | 答案，revision + 1 |
| Re-ask | `POST /api/inbox/messages/:id/reask` | 一条 person 备注（`reask`：该会话的 open 问题）作为对该消息的回复；该会话未发送的 pick 随之标记为已发送。该会话无 open 问题时拒绝（`nothing_to_reask`），对 person 的消息拒绝，在归档线之前拒绝。按其 key 幂等（回复的 key：`idempotency_key_reused`）。规则 2a 在每个 open 问题都位于一个等待中的备注时成立：另一会话的新问题、最后列出的问题被回答、superseded 或 closed，或备注所指会话再次写入，都会结束它 |
| Send | `POST /api/inbox/messages/:id/reply` | 列出的 pick 作为已发送（在该消息上，或带 `message_id` 在同一线程和会话的另一消息上），然后一条 person 回复。按其 key 幂等 |
| 执行跳过的 Send | 同上，`skip: [keys]` | 还将每个列出的仍为 open 的问题作为已发送的跳过（`by_reply`）。仅限 person 用 "Skip them" 选择的内容；Send 不跳过其他任何内容，且 `skip_unanswered` 被忽略（批准 "inbox-explicit-skip"） |
| Mark read | `POST /api/inbox/threads/:id/read` | 根消息上的已读字段和查看；对每个等待中的问题写入 `set_aside` |
| Undo | `POST /api/inbox/threads/:id/unread` | 清除已读字段，在已读有效期间恢复查看，恢复 Mark read 关闭的问题；对其中已告知 agent 的问题，每条提问消息一条重新打开备注。根消息无 Mark read 时不写入 |
| Mark unread | `POST /api/inbox/threads/:id/mark-unread` | 将根消息上的查看设为恰在最新 agent 消息之前（它再次未查看：New）；不写入其他内容，因此有效的 Mark read 保持，Move back 仍重新打开它关闭的内容。有未查看新闻的有效 Mark read 位于 New；下一次查看将其放回 Quiet，Mark read 在不产生新 Mark read 的情况下恢复查看。无 agent 写入、person 的真实回复在最新 agent 消息之后，或它已未查看时不写入 |
| 搁置备注 | 服务器，Mark read 后 10 秒（以及 Unarchive 后） | 每条提问消息一条 person 备注，指明 Mark read 关闭且尚无备注指明的问题，无论此后发生了什么。被归档关闭的消息不产生备注 |
| Archive | `POST /api/inbox/messages/:id/resolve` `{ resolved: true }` | `resolved_at`。已 resolved 时不写入 |
| Unarchive | 同上，`{ resolved: false }` | 清除 `resolved_at`；在归档线之后做出的 Mark read 在同一根消息行中清除 |
| Agent 读取你的回复 | `read_thread` | `agent_checked_seq` |
| 全部标记为已读（一个项目） | `POST /api/inbox/projects/:id/read` | 对项目中提供单次 Mark read 的每个线程（open 或 come back，有内容等待 person）执行 Mark read：恰好是每个单次 Mark read 写入的行。每个线程的搁置备注各自进行 |
| 全部归档（一个项目） | `POST /api/inbox/projects/:id/resolve` `{ resolved: true }` | 对项目中每个列出的 open 线程执行 Archive；对 came back 的线程执行 Mark read。恰好是单次操作的行 |
| 它们的 Undo | `…/unread` `{ threads }`、`…/resolve` `{ resolved: false, threads }` | 对批次指定的确切线程执行 Undo 或 Unarchive，如单次操作所写入 |
| Person 发起线程 | `POST /api/inbox/messages` | 一个带 `to` 和 `started_by_person` 的 person 根消息；无论正文说什么，都没有问题记录。按其 key 幂等。会话不活跃时拒绝（`session_not_live`），不写入任何内容 |
| Delivered | 桥接的 `delivered` 事件 | person 消息上的 `delivery`，以及截至它的 `agent_checked_seq` |
| 回复草稿 | `POST /api/inbox/threads/:id/draft` `{ body, base }`；`…/draft/delete`；Send 上的 `clear_draft` | 草稿的文件（无行）。空正文删除它。过期的 `base` 为 `409 draft_conflict`，不写入任何内容 |

## 转换表

列是事件之前的状态：**Asks**（Stopped、Holding 或 Waiting）、**New**、**Sent**、**Read**（由 Mark read 归入 Quiet）、**Quiet**、**Archived**、**Came back**。单元格指明之后的状态。"Asks" 指最强 open 问题所在的分区。**Archived** 涵盖两种情况，它们对 person 的写入回答不同：归档线之后无 agent 写入（每次写入都被拒绝，`409 thread_resolved`），以及一个 came back 并在处理后返回归档的线程（其较晚消息表现得像 open 线程的；较早消息保持 closed）。已归档线程没有 open 或 picked 问题，