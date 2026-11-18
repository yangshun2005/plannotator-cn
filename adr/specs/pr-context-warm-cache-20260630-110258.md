# 规格：PR 上下文预热缓存

日期：2026-06-30

## 意图

在审查服务器中预加载 PR Overview 上下文，使 Overview 面板通常能直接拿到已获取的描述、评论、审查线程、检查、标签、合并状态和关联 Issue 数据。

## 范围

修改：

- `packages/server/review.ts`
- `apps/pi-extension/server/serverReview.ts`

不修改 React 客户端。

## 行为

PR 审查服务器启动且初始 PR ref 可用时，服务器在后台开始获取 `fetchPRContext(prRef)`。

后台获取按 PR URL 缓存。

请求 `/api/pr-context` 时，处理器返回缓存结果。若缓存中的获取仍在运行，处理器等待同一个 promise。

若当前 PR 没有缓存获取，`/api/pr-context` 会启动一个、存入缓存、等待并返回。

用户通过 `/api/pr-switch` 切换 PR 时，服务器在拿到新 `prRef` 后预热新 PR URL 的上下文缓存。预热不得延迟切换响应。

若获取失败，在返回错误前从缓存中删除该 PR URL。后续请求可重试。

## API 契约

`GET /api/pr-context` 成功时继续返回现有的 `PRContext` JSON。

失败时继续返回 `{ error: string }`，状态码 500。

非 PR 模式继续返回 `{ error: "Not in PR mode" }`，状态码 400。

## 实现说明

添加会话级缓存：

```ts
const prContextCache = new Map<string, Promise<PRContext>>();
```

添加本地辅助函数：

```ts
const getCachedPRContext = (url: string, ref: PRRef): Promise<PRContext> => {
  const cached = prContextCache.get(url);
  if (cached) return cached;

  const promise = fetchPRContext(ref).catch((error: unknown) => {
    prContextCache.delete(url);
    throw error;
  });
  prContextCache.set(url, promise);
  return promise;
};
```

类型导入在不同运行时可能不同：

- 主服务器可从 `./pr` 导入 `PRContext` 和 `PRRef`（若尚未导入）。
- Pi 服务器可从 `../generated/pr-types.js` 导入。

启动预热：

```ts
if (prRef && prMetadata) {
  void getCachedPRContext(prMetadata.url, prRef).catch(() => {});
}
```

切换预热：

```ts
void getCachedPRContext(pr.metadata.url, prRef).catch(() => {});
```

端点：

```ts
if (!isPRMode || !prRef || !prMetadata) {
  return Response.json({ error: "Not in PR mode" }, { status: 400 });
}

const context = await getCachedPRContext(prMetadata.url, prRef);
return Response.json(context);
```

Pi 实现中使用 Pi 服务器的 `json(res, ...)` 辅助函数。

## 验证

运行：

```bash
bun run typecheck
```

可选的针对性手动检查：

1. 启动 PR 审查。
2. 立即打开 PR Overview。
3. 确认 `/api/pr-context` 正常返回。
4. 切换到另一个 PR。
5. 确认新 PR Overview 仍能加载，且 provider 获取失败时可重试。

## 风险

若审查服务器打开期间 PR 评论或检查发生变化，会话缓存的上下文可能过期。这符合需求行为，也与现有会话缓存风格一致。

provider 调用较慢时，预加载仍可能在 Overview 挂载前未完成。此时 UI 仍显示加载中，但会等待已在运行的获取，而非再启动一次。

对于用户从不打开 Overview 上下文的 PR 审查，服务器会多一次 provider 调用。这可接受，因为 PR Overview 现在默认打开，且该调用是只读的。