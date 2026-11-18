# 规格：源文件编辑竞态与冲突恢复

日期：2026-06-18

## 目标

让源文件的磁盘感知足够可靠，以便合并 PR #936。

具体来说：

- 不让过期的异步 `/api/doc` 响应覆盖更新的源文件状态
- 保存后不因较旧的协调完成较晚而丢失已保存的编辑上下文
- 除非 Plannotator 拥有恢复所需的磁盘文本，否则不进入可恢复的冲突状态

## 非目标

不实现：

- 已删除/重命名文件的恢复 UX
- 对已不存在文件的孤立草稿救援
- 在已保存编辑上下文未经验证的情况下允许提交反馈
- 更大规模的编辑器重写
- 编辑模式之外的背景覆盖路径

## 决策

### 1. 用请求顺序和起始哈希保护 watcher 协调

问题：

`reconcileOpenSourceDocuments` 可能启动一个 `/api/doc` 请求，然后在更新的状态已经落地之后才应用它。

决策：

每个源文件快照请求必须携带两项本地客户端状态：

- 每个文件的请求序列号
- 请求启动时该文档已知的磁盘哈希

请求返回时，仅在以下条件全部满足时才应用：

- 它仍是该文件的最新请求
- 该文档仍存在于 `editableDocuments` 中
- 该文档当前未在保存
- 该文档当前已知的磁盘哈希仍等于起始哈希

已知磁盘哈希指：

1. 若已知冲突，则为 `diskConflict.sourceSave.hash`
2. 否则若启用了 source-save，则为 `sourceSave.hash`
3. 否则为 `lastKnownHash`

`packages/editor/App.tsx` 中的实现形态：

```ts
const sourceReconcileSeqRef = useRef<Map<string, number>>(new Map());

function getKnownDiskHash(record: EditableDocumentRecord | null): string | undefined {
  return record?.diskConflict?.sourceSave.hash
    ?? (record?.sourceSave?.enabled ? record.sourceSave.hash : record?.lastKnownHash);
}

// inside reconcileOpenSourceDocuments loop
const startRecord = editableDocuments.getDocument(doc.key);
const expectedHash = getKnownDiskHash(startRecord);
const seq = (sourceReconcileSeqRef.current.get(doc.key) ?? 0) + 1;
sourceReconcileSeqRef.current.set(doc.key, seq);

const snapshot = await fetchSourceDocumentSnapshot(doc);
if (!snapshot) continue;
if (sourceReconcileSeqRef.current.get(doc.key) !== seq) continue;

const currentRecord = editableDocuments.getDocument(doc.key);
if (!currentRecord?.sourceSave?.enabled) continue;
if (currentRecord.saveStatus === 'saving') continue;
if (getKnownDiskHash(currentRecord) !== expectedHash) continue;

const result = editableDocuments.reconcileDiskSnapshot(...);
```

说明：

- 序列号守卫处理重叠的外部变更请求。
- 起始哈希守卫处理较旧请求在途时保存完成的情况。
- 两者都需要。
- 现有的"保存时跳过"守卫保留。

### 2. 从 `/api/source/save` 返回冲突快照

问题：

保存端点通过读取当前文件快照来检测冲突，但响应只返回哈希元数据。客户端随后执行第二次 `/api/doc` 读取以获取当前文本。如果该读取失败，客户端无法展示真正的冲突选择。

决策：

扩展 `SourceSaveResponse` 的冲突变体，使其包含当前磁盘快照。

`packages/shared/source-save.ts` 中的实现形态：

```ts
| {
    ok: false;
    code: "conflict";
    message: string;
    currentText: string;
    currentHash: string;
    currentMtimeMs: number;
    currentSize: number;
    currentEol: SourceFileEol;
  }
```

其他失败变体保持独立。它们不需要快照字段。

`packages/shared/source-save-node.ts` 中的实现形态：

```ts
if (before.hash !== baseHash) {
  return {
    ok: false,
    code: "conflict",
    message: "The file changed on disk since Plannotator opened it.",
    currentText: before.text,
    currentHash: before.hash,
    currentMtimeMs: before.mtimeMs,
    currentSize: before.size,
    currentEol: before.eol,
  };
}
```

这会自动同时适用于 Bun 和 Pi，因为两个服务器都调用共享的 `saveSourceFileAtomic`。

### 3. 直接应用保存冲突快照

问题：

`handleSaveEditedSourceFile` 目前通过调用 `reconcileOpenSourceDocuments` 来处理保存冲突，这会创建第二次 `/api/doc` 请求。

决策：

在收到 `409 conflict` 保存响应时，用活动文档的 source-save 元数据加上冲突载荷构建一个启用的 `sourceSave` 快照，然后直接调用 `editableDocuments.reconcileDiskSnapshot`。

`packages/editor/App.tsx` 中的实现形态：

```ts
if (!data.ok && data.code === 'conflict') {
  if (typeof data.currentText === 'string') {
    const conflictSourceSave = {
      ...activeSourceSave,
      hash: data.currentHash,
      mtimeMs: data.currentMtimeMs,
      size: data.currentSize,
      eol: data.currentEol,
    };

    const result = editableDocuments.reconcileDiskSnapshot({
      key: activeDocument.key,
      text: data.currentText,
      sourceSave: conflictSourceSave,
    });

    // update edit stats/toast like the current conflict path
  } else {
    editableDocuments.markError(activeDocument.key, message);
    toast.error('File changed on disk', {
      description: 'Plannotator could not load the latest disk version. Try saving again.',
    });
  }
  return true;
}
```

说明：

- 不再从保存冲突分支调用 `reconcileOpenSourceDocuments`。
- 除非存在 `diskConflict` 快照，否则不调用 `markConflict`。
- 可恢复冲突意味着我们同时拥有用户文本和磁盘文本。

### 4. 将非阻塞性发现排除在本补丁之外

不修改：

- 针对缺失/已删除源文件的已恢复脏草稿
- 已保存编辑上下文未经验证时的失败关闭提交行为

可选：

- Bun/Pi 的 source-save 资格判定谓词可以稍后提取。本补丁不需要。

## 测试

添加/更新针对性测试：

### 共享 source save

在 `packages/shared/source-save-node.test.ts` 中：

- 冲突响应包含 `currentText`
- 冲突响应包含 `currentHash`、`currentMtimeMs`、`currentSize`、`currentEol`
- 文件内容仍未被覆盖

### 可编辑文档协调

在 `packages/editor/editableDocuments.test.ts` 或一个小型辅助测试中：

- 已知磁盘哈希优先使用 `diskConflict.sourceSave.hash`
- 过期的预期哈希快照被跳过，而非应用

如果过期快照守卫完全保留在 `App.tsx` 本地，不要仅为它编造一个大型 App 测试装置。只有当提取微小的"已知哈希"辅助函数能让守卫可测试且无需引入整个应用时，才优先这样做。

### 现有测试

运行：

```bash
bun test packages/shared/source-save-node.test.ts packages/editor/editableDocuments.test.ts packages/editor/editableDocumentsHook.test.tsx packages/editor/savedFileChangeValidation.test.ts
DOM_TESTS=1 bun test --preload ./packages/ui/test-setup/happy-dom.ts packages/editor/editableDocumentsHook.test.tsx
bun run typecheck
bun test
```

## 手动验证

保存冲突：

1. 打开一个由源文件支持的文件夹文件。
2. 在 Plannotator 中开始编辑它。
3. 在外部修改同一文件。
4. 点击保存。
5. 确认冲突横幅出现，且不依赖第二次文件请求。
6. 确认 Reload 使用磁盘文本。
7. 确认编辑时 Overwrite 会保存 Plannotator 文本并记录已保存的 Edits 卡片。

过期 watcher：

1. 打开一个由源文件支持的文件。
2. 大约同时触发磁盘协调和保存。
3. 确认已保存文本和已保存 Edits 卡片不会被较旧的 watcher 响应回退。

第二个手动用例对时序敏感；自动化守卫是主要保护手段。