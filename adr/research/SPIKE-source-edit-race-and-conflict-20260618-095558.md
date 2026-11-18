# Spike：源文件编辑竞态与冲突恢复

日期：2026-06-18

## 问题

审查最新 PR #936 的发现，决定合并前实际应修复哪些问题。

主要关注点：

- 异步文件监听协调可能应用过期的磁盘快照
- 保存冲突可能进入冲突状态，但数据不足以恢复
- 缺失文件的已恢复草稿可能不可见
- 未验证的已保存编辑会阻止反馈提交
- Bun/Pi 重复了源文件保存资格判定逻辑

## 范围

已检查代码：

- `packages/editor/App.tsx`
- `packages/editor/editableDocuments.ts`
- `packages/editor/editableDocuments.test.ts`
- `packages/editor/editableDocumentsHook.test.tsx`
- `packages/shared/source-save.ts`
- `packages/shared/source-save-node.ts`
- `packages/shared/source-save-node.test.ts`
- `packages/server/annotate.ts`
- `apps/pi-extension/server/serverAnnotate.ts`
- `packages/server/reference-handlers.ts`
- `apps/pi-extension/server/reference.ts`

本次 spike 未修改任何产品代码。

## 发现

### 1. 乱序磁盘快照可能覆盖较新状态

该发现有效且重要。

当前流程：

1. SSE 文件监听事件调用 `reconcileOpenSourceDocuments`。
2. 对每个打开的源文档，获取 `/api/doc`。
3. 获取返回后，将返回的文本和 `sourceSave` 快照应用到 `editableDocuments.reconcileDiskSnapshot`。

当前守卫仅在文档被标记为 `saving` 时跳过。

这不够。一次获取可能在保存之前开始，在保存完成后才结束。此时文档已不再是 `saving`，而是 `saved`。过期的获取结果随后会被当作最新磁盘真相应用。

坏结果：

- 活动文档可能回退到较旧的磁盘文本
- 已保存编辑卡片可能被清除
- 反馈可能丢失已保存编辑上下文
- 下次保存可能命中假冲突，因为客户端已被移回旧的基础哈希

同一类 bug 也存在于两次快速外部写入：如果请求乱序完成，较旧的快照可能胜出。

为什么一个守卫不够：

- 仅有序号可防止旧请求胜过较新的协调请求，但不能阻止单个旧请求在用户保存后应用。
- 仅起始哈希检查可防止在文档已推进后应用，但如果较旧的快照先应用，可能丢弃两个重叠外部快照中较新的那个。

最佳修复：

- 同时使用每文件最新请求序号和每请求预期已知磁盘哈希
- 仅当该快照仍是该文件的最新请求，且文档的已知磁盘哈希仍是请求开始时当前的哈希时，才应用该快照

改动小，直接解决竞态。

### 2. 保存冲突恢复依赖第二次 `/api/doc` 请求

该发现有效。

当前流程：

1. 保存发送 `/api/source/save`。
2. 服务端调用 `saveSourceFileAtomic`。
3. 如果当前磁盘哈希与 `baseHash` 不匹配，服务端返回 `409 conflict` 及当前哈希元数据。
4. 客户端随后调用 `reconcileOpenSourceDocuments(..., { includeSavingKey })`。
5. 该调用请求 `/api/doc` 获取当前磁盘文本，以便 UI 填充 `diskConflict`。

如果后续 `/api/doc` 请求失败，客户端当前调用 `markConflict`，设置 `saveStatus: "conflict"` 但不填充 `diskConflict`。

这是坏状态：

- 横幅依赖 `diskConflict`，因此操作不显示
- 保存按钮看到冲突状态
- 重试继续使用过期的基础哈希

最佳修复：

不要依赖第二次请求。服务端在 `saveSourceFileAtomic` 内检测到冲突时已持有当前文件快照。在 `409` 响应中返回该冲突快照。

然后客户端可直接从保存响应构建冲突状态：

- 当前文本
- 当前哈希
- 当前 mtime
- 当前大小
- 当前 EOL

如果响应缺少该快照，将文档标记为错误，而非可恢复的磁盘冲突。可恢复冲突需要磁盘文本。

这比让后续获取更可靠更干净。

### 3. 源文件缺失时已恢复草稿可能不可见

该发现有效，但本轮不在范围内。

草稿恢复路径即使源文件不再正常打开，也可能重建 `editableDocuments` 记录。如果没有活动可编辑文档，这些恢复的编辑可能不会立即显示。

这与已删除/不可读文件恢复重叠，我们已明确决定暂不构建。

为何现在不修复：

- 它与过期快照损坏不同
- 需要产品决策：Plannotator 应显示孤立源编辑、允许保存为重建，还是仅让用户复制/丢弃？
- 添加快速自动打开行为可能在已删除文件周围产生混乱状态

建议：

留待单独的已删除/不可读源文件 UX 处理。

### 4. 未验证的已保存编辑阻止反馈提交

该发现有效但有意为之。

如果存在已保存文件编辑，且 `/api/doc` 无法验证其是否仍为当前，提交路径会阻止并提示用户重试。

这在瞬时故障期间令人烦恼，但比向 agent 发送过期或虚假编辑上下文更安全。

建议：

目前保持失败关闭行为。

### 5. Bun/Pi 重复源文件保存资格判定

该发现有效但属小问题。

单文件标注会话能否暴露源文件保存行为的判定同时出现在 Bun 和 Pi 服务端代码中。

目前一致。将其提取到 `packages/shared/source-save.ts` 可降低漂移风险，但与竞态或保存冲突损坏无关。

建议：

可选清理，不是本次修复的阻塞项。

## 建议

合并前修复：

1. 为源文档协调添加单调守卫。
2. 从 `/api/source/save` 返回冲突快照并直接应用。
3. 如果保存冲突响应无法提供当前磁盘文本，标记错误而非冲突。

本轮不修复：

- 已删除/不可读恢复草稿的救援
- 未验证提交的失败关闭行为
- Bun/Pi 判定清理，除非几乎零成本顺带完成