# 综合方案：PR 上下文预热缓存

日期：2026-06-30

## 摘要

最简单的正确改动只涉及服务端。客户端已经通过一个 Hook 请求 PR 上下文，并且已经按 PR URL 重置。缺失的一环是：服务端要等到该请求到达后才开始调用缓慢的 provider。

改动应在两个 review 服务端各加一个小型 promise 缓存。缓存以 PR URL 为键，存储进行中或已完成的 `fetchPRContext` promise。启动和 PR 切换时应预热缓存，但不等待它完成。`/api/pr-context` 处理器应等待缓存中的 promise，仅在没有预加载时才创建它。

## 决策压力

预加载必须减少 Overview 的加载闪烁，同时不延迟服务端就绪。这意味着启动调用必须分离但受控。把 promise 存进 map 就足以实现这种受控，因为端点可以等待同一份工作，且拒绝路径可以清除失败的 promise。

失败处理很重要。当 `gh pr view` 失败时，GitHub 可能让整个上下文获取失败。如果失败的 promise 留在缓存中，UI 的重试按钮会一直拿到同样的失败。在拒绝时清除可保留当前的重试行为。

缓存暂不应移入共享代码。两个服务端实现相似，但周边状态是局部的且与运行时相关。每个文件里放一个小辅助函数能让改动保持狭窄，避免引入没有更广泛复用价值的新抽象。

## 推荐形态

在两个文件中使用以下局部模式：

```ts
const prContextCache = new Map<string, Promise<PRContext>>();

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

使用 wrapper 导入中已有的服务端本地 PR ref 类型。如果导入 `PRRef` 在 Pi 中会增加摩擦，则使用 `../generated/pr-types.js` 的生成类型导入。

启动时不等待地调用：

```ts
void getCachedPRContext(prMetadata.url, prRef).catch(() => {});
```

PR 切换更新 `prRef` 后不等待地调用：

```ts
void getCachedPRContext(pr.metadata.url, prRef).catch(() => {});
```

在 `/api/pr-context` 中等待调用：

```ts
const context = await getCachedPRContext(prMetadata.url, prRef);
```

## 不要改动的内容

不要改动客户端 Hook 或 Overview 面板。

不要改动 `/api/pr-context` 的响应结构。

不要添加跨服务端会话的持久化。

不要引入 TTL，除非用户认为新鲜度比会话一致性更重要。

不要在同一次改动中重构 stack-tree 缓存。