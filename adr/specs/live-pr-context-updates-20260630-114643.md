# 规格：PR 上下文实时更新

日期：2026-06-30

## 意图

在 PR 审查打开期间，自动保持 PR Overview 上下文为最新，无需手动刷新、不重复读取 provider、UI 关闭后不做无谓的后台工作。

## 范围

更新两个服务端实现：

- `packages/server/review.ts`
- `apps/pi-extension/server/serverReview.ts`

更新审查客户端 Hook：

- `packages/review-editor/hooks/usePRContext.ts`

在有用处添加共享类型/辅助函数，但不要将其扩展为通用审查事件系统，除非本地实现出现重复或难以理解。

## 行为

PR 审查服务启动时，为初始 PR 预热 PR 上下文，不阻塞启动。

用户切换 PR 时，为新 PR 预热 PR 上下文，不阻塞切换响应。

PR 模式的浏览器标签页打开时，客户端订阅 PR 上下文实时更新。这应在 PR 模式的应用层完成，而不是仅在 Overview 面板挂载后。

服务端每 30 秒刷新一次被监视的 PR 上下文。

服务端对每个 PR URL 同时最多执行一次刷新。

服务端对每个 PR URL 刷新一次，而不是每个浏览器标签页刷新一次。

没有标签页监视某个 PR URL 时，服务端停止该 URL 的循环刷新定时器。

`/api/pr-action` 成功发布审查/评论后，服务端立即刷新目标 PR 上下文并广播更新后的上下文。

新上下文到达时 UI 自动更新。

如果流在快照到达前失败，客户端回退到自动轮询。用户无需按刷新。

## API 契约

保持 `GET /api/pr-context` 兼容：

- 成功：原始 `PRContext`
- 失败：`{ error: string }`
- 非 PR 模式：`{ error: "Not in PR mode" }`

新增流端点：

```text
GET /api/pr-context/stream
```

流发送 JSON 事件。建议的事件结构：

```ts
type PRContextStreamEvent =
  | { type: "snapshot"; url: string; version: number; context: PRContext | null; loading: boolean; error: string | null; stale: boolean }
  | { type: "updated"; url: string; version: number; context: PRContext; stale: false }
  | { type: "loading"; url: string; version: number }
  | { type: "error"; url: string; version: number; error: string; stale: boolean; retryAt?: number };
```

如果客户端和服务端都需要，确切的 TypeScript 类型可放在共享代码中。

## 服务端设计

将当前仅基于 Promise 的 PR 上下文缓存替换为实时缓存条目：

```ts
type PRContextCacheEntry = {
  url: string;
  ref: PRRef;
  context: PRContext | null;
  error: string | null;
  version: number;
  inflight: Promise<PRContext> | null;
  watchers: number;
  timer: ReturnType<typeof setTimeout> | null;
  nextAllowedRefreshAt: number;
};
```

使用 `ensurePRContextEntry(url, ref)` 创建或更新条目。

使用 `refreshPRContext(url, ref, reason)` 执行：

1. 若存在进行中的 Promise，返回该 Promise。
2. 遵守 `nextAllowedRefreshAt`。
3. 设置 `inflight`。
4. 广播 `loading`。
5. 调用 `fetchPRContext(ref)`。
6. 存储 `context`，清除 `error`，递增 `version`，广播 `updated`。
7. 失败时存储 `error`，递增 `version`，若被限流则设置冷却时间，广播 `error`。
8. 清除 `inflight`。

Provider 适配器不得将失败的主上下文读取转为成功的空上下文。GitHub 在 `gh pr view` 失败时已抛出异常。GitLab 目前容忍部分失败，但主 MR 详情调用必须视为实时刷新的必需项；否则限流或认证失败可能被广播为空白 PR 数据。

SSE 连接打开时使用 `startWatchingPRContext(url, ref)`：

1. 递增 watchers。
2. 发送快照。
3. 如需要，启动循环定时器。
4. 若无上下文且无进行中的刷新，触发立即刷新。

SSE 连接关闭时使用 `stopWatchingPRContext(url)`：

1. 递减 watchers。
2. watchers 归零时停止定时器。

被监视的正常刷新使用 30 秒间隔。

当 provider 看似被限流但无重置时间时，使用较短的保守冷却时间，例如 60 秒。

如果 provider 输出暴露了重试时间，使用该时间而非默认冷却时间。

## 客户端设计

修改 `usePRContext`，使其在 `prMetadata` 存在时打开 `EventSource("/api/pr-context/stream")`。

Hook 应继续返回：

```ts
{
  prContext,
  isLoading,
  error,
  fetchContext
}
```

`fetchContext` 应保留以兼容和显式重试按钮，但正常操作不应依赖它。

PR URL 变化时，重置本地状态并重新连接。

如果 SSE 在首个快照前失败，启动对 `/api/pr-context` 的自动轮询。轮询应基于服务端缓存，且不应导致重复的 provider 读取。

如果 SSE 之后重连，优先使用流式状态。

## 限流行为

不要主动轮询限流端点。那会消耗更多 provider 预算。

在 provider 命令失败前使用正常刷新节奏。

如果 GitHub 或 GitLab 返回限流错误：

- 尽可能保持最后一次成功的上下文可见。
- 显示小的过期/错误状态。
- 在重试时间或冷却时间过去前停止刷新。
- 自动重试。

写入应保持用户触发。实时刷新循环不得自行重新发布或重试写操作。

## 双向评论路径

本规格不实现完整的双向评论。

它通过让服务端拥有最新的远程上下文，并在写入后立即刷新，为此做准备。

未来的写入工作应添加 provider 特定端点：

- 通用 PR/MR 评论。
- 内联线程回复。
- 在支持处解决/取消解决讨论线程。
- 在支持处编辑/删除评论。

未来的客户端状态应使用本地 ID 跟踪待处理的本地评论，然后在服务端刷新上下文后将其与 provider ID 对账。

## 验证

运行：

```bash
bun run typecheck
bun test packages/server/review-workspace.test.ts apps/pi-extension/server.test.ts
```

在可行处添加针对性测试：

- 多个 `/api/pr-context` 请求共享一次进行中的刷新。
- 同一 PR 上的多个流监视者不会创建重复的 provider 刷新。
- 断开所有流监视者会停止循环定时器。
- 成功的 `/api/pr-action` 触发立即上下文刷新。
- provider 刷新失败保持可重试并广播错误。

手动检查：

1. 打开 PR 审查，确认 Overview 上下文无需手动操作即可出现。
2. 保持 PR 打开，从 GitHub/GitLab 添加远程评论；确认其自动出现。
3. 发布 Plannotator PR 审查；确认发布的内容自动出现。
4. 在两个标签页中打开同一审查；确认 provider 刷新不重复。
5. 切换 PR；确认新 PR 接收实时上下文更新。