# 规格说明：源文件编辑可靠性修复

日期：2026-06-18

## 目标

修复 PR #936 中剩余的源文件编辑可靠性问题，不将功能扩展为更大规模的编辑器重写。

实现应小而直接，并与现有源文件编辑架构保持一致：

- 源文件存放在 `useEditableDocuments` 中
- 当前活动的源文档驱动 `displayedMarkdown`
- 源文件磁盘变更通过现有的文件监听 SSE 流传入
- 标注重映射使用 `applyEditedDocument`

## 非目标

不要实现：

- 新的已删除/重命名文件恢复流程
- 编辑模式之外的背景覆盖系统
- 新的文件创建流程
- 新的标注基础设施
- 对 `useEditableDocuments` 的大范围重写

## 变更

### 1. 保持源文件监听流稳定

问题：

源文件监听的 `EventSource` 会在普通渲染中被销毁并重建，因为该 effect 依赖 `reconcileOpenSourceDocuments`，而后者依赖每次渲染返回的新 `editableDocuments` 对象。

决策：

让 `EventSource` effect 仅依赖 `sourceWatchDirsKey`。将最新的 reconcile 函数存入 ref。

`packages/editor/App.tsx` 中的实现形态：

```ts
const reconcileOpenSourceDocumentsRef = useRef(reconcileOpenSourceDocuments);

useEffect(() => {
  reconcileOpenSourceDocumentsRef.current = reconcileOpenSourceDocuments;
}, [reconcileOpenSourceDocuments]);

useEffect(() => {
  if (!sourceWatchDirsKey || typeof EventSource === 'undefined') return;

  // create EventSource and timers
  // ...
  timers.set(key, setTimeout(() => {
    timers.delete(key);
    void reconcileOpenSourceDocumentsRef.current(dir);
  }, 120));

  return () => {
    for (const timer of timers.values()) clearTimeout(timer);
    source.close();
  };
}, [sourceWatchDirsKey]);
```

说明：

- 这是标准的"稳定订阅、最新回调"模式。
- 不要仅为这一点而 memoize 整个 editable-documents API。
- 流应在被监听目录集合变化时重连，而不是在无关 UI 状态变化时重连。

验证：

- 类型检查。
- 手动/针对性检查：打开一个由源文件支持的 annotate 文件夹文件，触发普通 UI 渲染，确认在被监听目录集合变化之前只创建一个活动的源文件监听流。

### 2. 将空的单文件 annotate 文件注册为可编辑

问题：

空的本地文件返回 `plan: ""`。客户端使用 `else if (data.plan)`，因此跳过了源文件编辑设置。

决策：

对非文件夹 markdown 文档使用字符串检查而非真值检查。

`packages/editor/App.tsx` 中的实现形态：

```ts
} else if (typeof data.plan === 'string') {
  const normalizedPlan = data.plan.replace(/\r\n?/g, '\n');
  setMarkdown(normalizedPlan);
  originalMarkdownRef.current = normalizedPlan;
  if (data.mode === 'annotate' && data.sourceSave?.enabled) {
    const key = editableDocumentKey(data.sourceSave, `file:${data.sourceSave.path}`);
    editableDocuments.openDocument({ key, text: normalizedPlan, sourceSave: data.sourceSave });
  }
}
```

将此分支保留在 `annotate-folder` 分支之后，以便文件夹模式仍以空白开始并等待文件选择。

验证：

- 如可行，围绕 API 初始化行为添加或更新一个针对性测试。
- 手动检查：`touch empty.md`，运行 `plannotator annotate empty.md`，确认编辑控件出现，且输入内容后保存有效。

### 3. 重新加载磁盘冲突内容时重映射标注

问题：

`handleReloadDiskConflict` 直接替换 markdown。这绕过了共享的标注重映射行为。

决策：

通过其他文档文本变更所使用的同一辅助函数执行重新加载。

`packages/editor/App.tsx` 中的实现形态：

```ts
const reloaded = editableDocuments.reloadDiskConflict(activeDocument.key);
if (!reloaded) return;

const remapped = applyEditedDocument(reloaded.currentText);
repaintHighlights(remapped);
editSessionBaseRef.current = reloaded.currentText;
setEditorDirty(false);
setEditorDiffersFromBaseline(false);
setEditStats(null);
scheduleDraftSave();
```

说明：

- 如果原始选中文本已不存在，标注保留在侧边栏中并失去高亮。
- 如果编辑模式仍处于打开状态，`applyEditedDocument` 会递增 `editGeneration`，从而以重新加载的磁盘文本重新挂载编辑器。
- 不要仅因标注文本无法高亮就删除标注。

验证：

- 现有的 Hook 级重新加载测试应仍然通过。
- 仅在已有轻量测试框架时添加 App 级测试；否则依赖类型检查加手动 UI 验证。
- 手动检查：创建一条标注，制造磁盘冲突，重新加载磁盘，确认过期高亮消失且侧边栏标注保留。

### 4. 让冲突横幅在编辑模式之外如实反映

问题：

即使 `isEditingMarkdown` 为 false，冲突横幅也可能显示"Overwrite disk"。该操作无法执行，因为源文件保存需要实时编辑器缓冲区。

决策：

不要构建单独的背景覆盖流程。仅在编辑时显示"Overwrite disk"。保留"Reload from disk"可用。

`packages/editor/App.tsx` 中的实现形态：

```tsx
{isEditingMarkdown && (
  <button type="button" onClick={handleOverwriteDiskConflict}>
    Overwrite disk
  </button>
)}
<button type="button" onClick={handleReloadDiskConflict}>
  Reload from disk
</button>
```

可选的文案调整：

- 编辑中："`filename` 在你编辑期间于磁盘上发生了变化。"
- 未编辑："`filename` 在磁盘上发生了变化。"

说明：

- 这符合产品模型：源文件编辑要么已保存、要么已丢弃、要么仍在编辑中。
- 如果用户未在编辑，覆盖不是一个合理的操作。

验证：

- 手动检查编辑模式下的冲突横幅：两个按钮均出现。
- 手动检查编辑模式之外的冲突横幅（如可触达）：仅显示重新加载。

## 测试计划

运行：

```bash
bun run typecheck
bun test packages/editor/editableDocuments.test.ts packages/editor/editableDocumentsHook.test.tsx packages/editor/savedFileChangeValidation.test.ts
```

如果添加了源文件监听测试覆盖，还需运行相关的 DOM 测试命令。

合并前，手动测试：

```bash
touch /tmp/plannotator-empty.md
plannotator annotate /tmp/plannotator-empty.md
```

然后输入内容、保存，确认文件写入磁盘。

同时手动测试源文件冲突：

1. 在 Plannotator 中打开一个文件夹文件。
2. 开始编辑。
3. 在外部修改同一文件。
4. 确认冲突横幅出现。
5. 确认重新加载使用磁盘文本且不留下错误高亮。
6. 确认覆盖仅在编辑器打开时出现。