# SPIKE：实时 PR 上下文更新

日期：2026-06-30

## 问题

Plannotator 应如何在不要求手动刷新的情况下，保持 PR 描述、评论、审查线程、检查和合并状态为最新，同时为双向 PR 评论保留一条清晰的路径？

## 发现

当前 PR Overview 通过一个客户端 Hook 读取上下文：`packages/review-editor/hooks/usePRContext.ts`。该 Hook 对每个 PR URL 只请求一次 `/api/pr-context`，将结果存入 React 状态，仅在出错后重试。它不订阅更新。

两个审查服务器现在都有预热的 PR 上下文 Promise 缓存：

- `packages/server/review.ts`
- `apps/pi-extension/server/serverReview.ts`

缓存以 PR URL 为键，在启动和切换 PR 时开始执行 `fetchPRContext(prRef)`，并让 `/api/pr-context` 等待缓存的 Promise，而不是发起重复工作。这消除了重复请求，但成功的上下文结果在会话内仍会过期。

当前 `/api/pr-context` 处理器只返回快照。没有版本字段、流或服务端刷新循环来处理 PR 上下文。

仓库已有可用的 SSE 模式：

- 外部注解使用 `/api/external-annotations/stream`，并在 `packages/ui/hooks/useExternalAnnotations.ts` 中有轮询回退。
- 代理任务使用 `/api/agents/jobs/stream`，并在 `packages/ui/hooks/useAgentJobs.ts` 中有轮询回退。
- 主服务器和 Pi 服务器都为长连接流禁用了空闲超时。

PR 写入路径在两个审查服务器中都是 `/api/pr-action`。它调用 `submitPRReview(...)`，然后返回 `{ ok: true, prUrl }`。成功发布后不会刷新或广播 PR 上下文。

提供方上下文成本有界但并非免费：

- GitHub 上下文当前运行 `gh pr view --json ...`，然后为审查线程执行一次 `gh api graphql` 调用。显式线程查询上限为 `reviewThreads(first: 100)` 和 `comments(first: 50)`。
- GitLab 上下文当前并行运行五次 `glab api` 调用，分别获取 MR 详情、备注、批准、流水线和关联 Issue。如果存在流水线，还会为流水线作业运行第六次调用。

提供方写入方式不同：

- GitHub 审查提交发送一个审查请求，可同时包含正文和内联评论。
- GitLab 提交将正文作为一条备注发布，然后将每个内联文件评论作为单独的讨论发布。

认证已委托给用户本地的提供方 CLI：

- GitHub 使用 `gh`。
- GitLab 使用 `glab`。

这意味着读写都以登录的 CLI 用户身份运行。此设计不需要单独的 Plannotator OAuth 流程。

## 速率限制说明

截至 2026-06-30，GitHub 文档说明普通用户令牌的认证主 REST 限制为每小时 5,000 次请求，GraphQL 主限制为每小时 5,000 点。GitHub 还记录了次级限制，如并发请求限制和内容创建限制。来源：https://docs.github.com/rest/using-the-rest-api/rate-limits-for-the-rest-api 和 https://docs.github.com/en/graphql/overview/rate-limits-and-query-limits-for-the-graphql-api

截至 2026-06-30，GitLab.com 文档说明每分钟 2,000 次认证 API 请求，Issue 和合并请求每分钟 60 次备注创建请求，以及项目作业端点限制。自管理 GitLab 实例可配置不同的限制。来源：https://docs.gitlab.com/user/gitlab_com/ 和 https://docs.gitlab.com/security/rate_limits/

30 秒刷新间隔对正常使用是可行的：

- GitHub：每小时约 120 次刷新，每次刷新约两次提供方操作。实际 GraphQL 成本因查询形状而异，但对于一个被监视的 PR，这远低于正常主限制。
- GitLab：一个被监视的 MR 每分钟约 10 到 12 次提供方请求，远低于 GitLab.com 正常的认证 API 限制。

主要的速率限制风险不是普通的后台读取。现实风险包括：多个打开的 PR 会话使用同一令牌、一个非常大的 PR 其 GitHub GraphQL 查询消耗更多点数、其他工具消耗同一用户令牌，以及 GitLab 因大量内联评论产生的写入突发。

## SSE 说明

SSE 不调用 GitHub 或 GitLab。它只保持浏览器到本地 Plannotator 服务器的连接打开。

实际的浏览器/服务器关注点是每个标签页打开的本地流数量。审查模式已经可以为外部注解和代理任务打开流。增加一个 PR 上下文流是可以接受的，但实现不应为摘要、评论、检查和写入分别添加独立的流。

如果以后流数量成为问题，PR 上下文事件可以合并到更广泛的审查事件流中。首次实时上下文实现不需要这样做。

## 建议方向

将 PR 上下文转变为服务器拥有的实时缓存。

服务器应为每个 PR URL 保留一个缓存条目。每个条目应存储最新上下文、最新错误（如有）、版本号、进行中的刷新 Promise、监视者数量、定时器状态，以及因速率限制或提供方错误导致的任何冷却。

启动和切换 PR 时仍应预热一次上下文。

当至少一个浏览器标签页正在监视某个 PR 时，服务器应按固定节奏刷新该 PR 上下文。从 30 秒开始。不要每个标签页刷新一次。不要在一个刷新已在进行中时启动新的刷新。

`/api/pr-action` 成功发布后，服务器应立即刷新目标 PR 上下文并广播结果。

客户端应在 PR 模式下自动订阅。正常新鲜度不应需要手动刷新按钮。如果 SSE 在快照到达前失败，使用与现有外部注解和代理任务 Hook 类似的轮询回退。

当某个 PR URL 没有监视者时，服务器应停止周期性刷新。成功的上下文可以在审查会话剩余时间内保持缓存。

提供方速率限制处理应保守：

- 如果提供方命令因明显的速率限制响应而失败，在重试前暂停该 PR 条目。
- 如果有重置时间或 retry-after 值，使用它。
- 如果没有，使用较短的保守冷却时间，如 60 秒，并继续显示上次成功的上下文，同时显示可见的过期/错误状态。

## 待定问题

`/api/pr-context` 应添加 `version` 包装，还是只有流使用带版本的封装，而现有快照端点保持原始 `PRContext`？保持现有端点原始可避免破坏当前 API。

流应在应用处于 PR 模式时始终活跃，还是仅在 PR Overview 面板打开时活跃？从用户体验看，应用级 PR 模式更好，因为 Overview 是默认视图，打开时上下文应已是最新。

首次实现是否应包含提供方特定的响应头解析？使用 `gh` 和 `glab` 时，当前命令调用并不总能获取响应头。首次实现可以处理明显的 CLI 速率限制错误，稍后添加更丰富的适配器元数据。

GitLab 内联评论写入是否需要排队或批处理？GitLab 当前将每个内联评论作为单独的讨论发布。排队待处理写入对未来的双向评论有用，但保持远程上下文新鲜并不需要。