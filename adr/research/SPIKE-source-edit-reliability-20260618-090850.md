# Spike：源文件编辑可靠性跟进

日期：2026-06-18

## 问题

在实施前验证 PR #936 剩余的审查发现：

- 源文件监听器重连过于频繁
- 空的 annotate 文件无法编辑
- 从磁盘重新加载后遗留过期的标注锚点
- 冲突横幅在无法执行覆盖时仍显示覆盖操作

## 范围

本 spike 仅阅读当前分支代码，不修改产品代码。

检查的文件：

- `packages/editor/App.tsx`
- `packages/editor/editableDocuments.ts`
- `packages/editor/editableDocuments.test.ts`
- `packages/editor/editableDocumentsHook.test.tsx`
- `packages/ui/components/MarkdownEditor.tsx`
- `packages/ui/hooks/useLinkedDoc.ts`
- `packages/server/annotate.ts`
- `apps/pi-extension/server/serverAnnotate.ts`

## 发现

### 1. 源文件监听流重连过于频繁

`App.tsx` 中的源文档监听器为包含已打开可编辑源文件的目录打开一个 `EventSource`。

监听 effect 依赖 `reconcileOpenSourceDocuments`。

`reconcileOpenSourceDocuments` 依赖 `editableDocuments`。

`useEditableDocuments()` 每次渲染都返回一个新的对象字面量。其方法大多是稳定的回调，但对象包装本身不稳定。

这意味着普通的 React 渲染会重建 `reconcileOpenSourceDocuments`，从而重新运行监听 effect。清理逻辑会关闭当前的 `EventSource` 并清除任何待执行的 120ms reconcile 定时器。

影响：

- 源文档打开期间反复重连 `/api/reference/files/stream`
- 待执行的磁盘变更 reconcile 工作可能被无关渲染取消
- 磁盘变更感知变得不如预期可靠

最佳小修复：

- 让实际流仅以 `sourceWatchDirsKey` 为键
- 将最新的 `reconcileOpenSourceDocuments` 回调存入 ref
- 收到 SSE 消息时，调用 `reconcileOpenSourceDocumentsRef.current(dir)`

这样连接保持稳定，同时仍调用最新逻辑。

已否决的较大修复：

- 对整个 `useEditableDocuments()` 返回对象做 memo 化范围更大，且无法完全解决此问题。该对象包含 `version`、`activeDocument`、`fileEditStatuses` 等状态，因此仍会因真正的可编辑文档更新而变化。流不应依赖它。

### 2. 空的单文件 annotate 目标无法变为可编辑

Bun 和 Pi 的 annotate 服务器对本地单文件 annotate 目标都返回 `plan: snapshot.text`。如果文件为空，服务器正确返回 `plan: ""` 以及已启用的 `sourceSave`。

`App.tsx` 中的客户端初始化路径仅在以下位置初始化普通文档状态：

```ts
} else if (data.plan) {
```

空字符串无法通过该检查。因此客户端不会：

- 将 `markdown` 设为空源文本
- 设置 `originalMarkdownRef`
- 调用 `editableDocuments.openDocument(...)`

编辑门控已支持空的源文件：

```ts
activeEditableDocument?.sourceSave?.enabled || displayedMarkdown !== '' || editStats !== null
```

所以服务器和编辑门控都已就绪；问题在于初始化条件。

影响：

- `plannotator annotate empty.md` 打开了一个有效的可编辑源文件，但客户端从未将其注册为可编辑
- 未来的"新建空文件"流程会遇到同样的问题

最佳小修复：

- 用 `typeof data.plan === 'string'` 替换真值判断
- 保持现有的 `annotate-folder` 分支独立，因为文件夹模式有意以未选择文件开始

### 3. 从磁盘重新加载绕过了标注重映射

普通文档编辑使用 `applyEditedDocument(next)` 和 `repaintHighlights(remapped)`。

该路径重新解析 markdown，并按原始选中文本重映射标注。如果文本已不存在，标注仍保留在侧边栏中，但失去其块锚点，且不会高亮随机文本。

`handleReloadDiskConflict` 目前改为执行：

```ts
setMarkdown(reloaded.currentText);
setEditGeneration((g) => g + 1);
```

这替换了文本，但跳过了共享的标注重映射/重绘路径。

影响：

- 冲突重新加载后，标注元数据仍可能指向旧块
- 导出的反馈可能携带过期的行/块上下文
- 高亮可能错误或出现不一致的失败

最佳小修复：

- 在 `editableDocuments.reloadDiskConflict(...)` 之后，调用 `applyEditedDocument(reloaded.currentText)`
- 然后调用 `repaintHighlights(remapped)`
- 保留现有的脏状态/统计重置和草稿保存行为

重要的编辑器细节：

`MarkdownEditor` 仅在挂载时读取 `markdown`。其 `documentId` 包含 `editGeneration`，因此如果编辑模式仍打开，`applyEditedDocument` 递增 `editGeneration` 就足以用重新加载的磁盘文本重新挂载编辑器。

### 4. 冲突横幅在无法执行覆盖时仍显示覆盖

`handleOverwriteDiskConflict` 调用 `handleSaveEditedSourceFile({ overwriteDiskConflict: true })`。

如果 `isEditingMarkdown` 为 false，`handleSaveEditedSourceFile` 会立即返回 `false`。

正常的预期源文件流程在按下 Done 后不会留下未保存的脏编辑。对于源文件，退出编辑控件是保存、丢弃或继续编辑。因此审查者所说的"Done 但未保存"路径并非预期的产品路径。

然而，代码仍可能通过更窄的路径在非编辑状态下产生冲突记录，例如：

- 冲突后丢弃本地编辑，但冲突对象仍保留
- 恢复或协调异常的草稿/冲突状态

在该状态下，显示"Overwrite disk"具有误导性，因为覆盖依赖实时编辑器缓冲区。

影响：

- 不是主要的数据丢失路径
- 仍是 UI 诚实性问题

最佳小修复：

- 仅在 `isEditingMarkdown` 为 true 时显示"Overwrite disk"
- 保持"Reload from disk"可用
- 目前不构建单独的后台覆盖路径

这符合产品规则：覆盖意味着"保留我正在编辑的版本"。

## 非发现

### 无法验证已保存编辑时阻止发送

这是保守但有意为之。如果 Plannotator 无法验证已保存的编辑上下文，它会阻止发送，而不是向 agent 发送过期或错误的上下文。

本轮不建议更改。

### 重新打开已恢复的文件夹草稿文件

这是审查中提出的，但不在本次请求的修复集内。它令人困惑但不会丢失数据：草稿文档记录已恢复并可通过文件树状态看到，但第一个恢复的文件不会自动打开。

除非以后自然出现极简方案，否则不建议更改。