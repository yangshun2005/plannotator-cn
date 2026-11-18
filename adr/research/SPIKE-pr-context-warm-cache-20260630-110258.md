# SPIKE: PR 上下文预热缓存

日期：2026-06-30

## 问题

Plannotator 应如何预加载 PR Overview 上下文，使 Overview 面板打开后不必等待一次全新的 `/api/pr-context` 请求？

## 发现

主审查服务器在 `packages/server/review.ts` 中维护 PR 模式的会话状态和缓存。它已有 30 秒的 PR 列表缓存、按 URL 的 PR 切换缓存，以及按 URL 的栈树缓存，位于 `packages/server/review.ts:219` 附近。

Pi 审查服务器在 `apps/pi-extension/server/serverReview.ts` 中采用相同结构。其 PR 列表缓存、切换缓存和栈树缓存位于 `apps/pi-extension/server/serverReview.ts:257` 附近。

两个运行时目前都在启动时获取栈树数据，并按 PR URL 存储解析结果。主服务器在 `packages/server/review.ts:813`；Pi 在 `apps/pi-extension/server/serverReview.ts:279`。当前代码会等待这次启动获取。期望的 PR 上下文行为仍应使用 Promise 值缓存，以便 UI 共享正在进行的预加载。

两个运行时目前都惰性获取 PR 上下文。主 `/api/pr-context` 处理器在 `packages/server/review.ts:1349` 直接调用 `fetchPRContext(prRef!)`。Pi 处理器在 `apps/pi-extension/server/serverReview.ts:1245` 做同样的事。

PR 切换会更新活动 PR 元数据和 `prRef`，使 PR 列表缓存失效，重新计算栈信息，获取或复用栈树数据，并返回新的 diff 载荷。主切换路径从 `packages/server/review.ts:1233` 开始；Pi 路径从 `apps/pi-extension/server/serverReview.ts:1138` 开始。这是为新激活的 PR 预热 PR 上下文的正确位置。

客户端已有一条 PR 上下文获取路径。`packages/review-editor/hooks/usePRContext.ts:23` 获取 `/api/pr-context`，按 PR URL 跟踪一个请求，并在 `prMetadata.url` 变化时重置。Overview 面板在挂载时于 `packages/review-editor/dock/panels/ReviewPROverviewPanel.tsx:94` 触发该获取。此服务端预加载无需客户端改动。

`fetchPRContext` 在两个服务器中都是运行时包装器。主包装器是 `packages/server/pr.ts:96`；Pi 包装器是 `apps/pi-extension/server/pr.ts:79`。

GitHub PR 上下文在 `gh pr view` 失败时可能抛出异常，位于 `packages/shared/pr-github.ts:338`。GitHub 审查线程在 GraphQL 失败时降级为空列表。GitLab 上下文在 `packages/shared/pr-gitlab.ts:244` 并行发起多个只读调用，大多将部分失败降级为空切片。

根测试脚本是 `bun test` 和 `bun run typecheck`，定义在 `package.json:35`。

## 约束

端点响应结构不得改变。UI 期望成功时返回原始 `PRContext` JSON，失败时返回 `{ error }`。

预加载不得阻塞服务器启动或 PR 切换响应。

缓存键应为 PR URL，与现有 `prSwitchCache` 和 `prStackTreeCache` 的键一致。

被拒绝的预热 Promise 应被移除，以便后续显式 UI 请求可以重试，而不是复用陈旧的失败。

两个服务器实现必须一起更新。

## 建议方向

为每个服务器实现添加一个按会话的 `Map<string, Promise<PRContext>>`。

添加本地 `getPRContextForUrl(url, ref)` 辅助函数，其行为如下：

1. 存在时返回现有 Promise。
2. 不存在时创建 `fetchPRContext(ref)`。
3. 立即存储该 Promise。
4. 如果 Promise 被拒绝，删除该映射条目。
5. 返回该 Promise。

启动时，如果 `prRef` 和元数据存在，调用该辅助函数但不等待它。

在 `/api/pr-context` 中，使用活动 PR URL 和活动 `prRef` 等待该辅助函数。

在 `/api/pr-switch` 中，更新活动 PR URL 和 `prRef` 后，为新 PR 调用该辅助函数但不等待它。

## 待解决问题

成功的上下文应缓存整个审查会话，还是应有 TTL？现有切换缓存和栈树缓存都是会话缓存，当前需求是沿用该行为。会话缓存是改动最小的方案。

启动时是否也应将栈树获取改为基于 Promise？这会使实际代码与所述模式一致，但对此 PR 上下文修复并非必要，且会扩大改动范围。