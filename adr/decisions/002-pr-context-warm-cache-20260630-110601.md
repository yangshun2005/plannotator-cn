# 002. 预热 PR 上下文缓存

日期：2026-06-30

## 状态

已接受

## 背景

PR Overview 面板展示 PR 描述、评论、评审线程、检查、标签、合并状态和关联 Issue。目前评审 UI 打开 Overview 后才向服务器请求 `/api/pr-context`。服务器要等该请求到达才调用 `fetchPRContext(prRef)`，因此面板经常闪现"Loading PR..."，同时 provider 命令还在运行。

主 Bun 评审服务器和 Pi 评审服务器已为 PR 列表、PR 切换载荷、栈树等相关数据维护 PR 模式会话缓存。客户端已有按 PR URL 索引的单一 PR 上下文获取路径，因此可在服务器端修复，无需改动 UI。

该获取不能阻塞服务器启动或 PR 切换响应。provider 失败也不能污染缓存，因为 UI 重试路径在瞬时失败后仍应能发起新请求。

## 决策

在两个评审服务器实现中增加会话级 PR 上下文 Promise 缓存：

- `packages/server/review.ts`
- `apps/pi-extension/server/serverReview.ts`

缓存以 PR URL 为键，存储 `Promise<PRContext>`。

PR 评审启动时，若已有初始 PR ref 和 URL，服务器在后台启动 `fetchPRContext(prRef)` 并存储该 Promise，不等待其完成。

请求 `/api/pr-context` 时，处理器等待当前 PR URL 对应的缓存 Promise。若获取仍在进行，处理器等待同一个 Promise。若无缓存 Promise，处理器创建并存储一个，等待其完成并返回结果。

`/api/pr-switch` 切换当前 PR 后，服务器在拿到新 `prRef` 后为新 PR URL 预热缓存。预热不延迟切换响应。

若 PR 上下文获取被拒绝，返回错误前会移除该 PR URL 的缓存条目。后续请求可重试。

`/api/pr-context` 的响应契约不变。

## 影响

UI 请求时 PR Overview 的上下文数据通常已就绪，"Loading PR..." 闪现基本消失。

若 UI 请求时 provider 获取仍在运行，UI 等待已启动的工作，不会触发重复的 provider 调用。

成功的 PR 上下文在评审服务器会话生命周期内缓存。若服务器保持打开期间评论或检查发生变化，缓存可能过期；目前接受此行为，符合所要求的会话缓存行为。

失败的获取仍可重试，因为被拒绝的 Promise 会从缓存中移除。

两个评审服务器实现必须保持此行为同步。