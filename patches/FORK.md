# 我们对 @pierre/trees 的 Fork

`@pierre/trees` 1.0.0（Apache-2.0，The Pierre Computer Company，github.com/pierrecomputer/pierre，标签 `@pierre/trees@1.0.0`），附带一个小补丁：`patches/@pierre%2Ftrees@1.0.0.patch`，由根 `package.json` 中 Bun 的 `patchedDependencies` 应用。该包自身的许可证和声明未改动；补丁只新增代码。

采用补丁而非 vendored 副本，是因为改动约 120 行新增代码、涉及 7 个构建文件，读起来就是普通 diff。全部为增量改动：不传新选项的树渲染结果与上游完全一致。

## 新增内容及原因

我们的审查树在行内放置控件和数字。上游的行只接受一个不可点击的装饰，因此新增：

| 新增 | 原因 |
|---|---|
| `rows.renderParts(context)` 返回 `{ lead, trail, muted, tone, title }` | `lead` 部分位于名称之前（已查看圆圈、git add），`trail` 部分位于名称之后（评论数、`+N`、`−M`）。`null` 部分保留空列以对齐。`muted` 使名称变暗（已查看文件）。`tone` 指定行样式（分组标题）。`title` 是行的工具提示 |
| 带 `id` 的部分是按钮（`role="button"`，`aria-pressed` 来自 `pressed`），调用 `rows.onPartClick({ partId, path, row })`；一个部分可同时带图标和文本 | 点击已查看圆圈或 git add 不得选中或打开文件；未关闭线程数是图标加数字 |
| `rows.onDoubleClick({ path, row })` | 双击固定文件 |
| `rows.onActivate({ path, row })` | 每次普通点击行以及按 Enter 或 Space（行是按钮）时调用，行已选中时也调用；Pierre 此时不触发选择变更，因此启动时点击高亮文件无反应。修饰键点击（范围或切换选择）不激活。移除条件：若上游新增激活事件，使用它并删除行 `onClick` 中的调用 |
| `rows.gitStatusPlacement: 'start'` | 状态字母位于名称之前，与我们的树一致 |
| `rows.nameTruncation: 'end'` | 名称作为单个字符串，末尾省略号（`row-name` 部分），按完整字母截断；Pierre 的中间截断会留下被切字母的碎片，且仍有空间的名称也被截断 |
| `rows.nameTruncation: 'keep-tail'` 配合 `rows.observeName(element, name, setText)` | 由宿主将文件名适配到行：补丁只渲染 `TailKeptName`（一个 `row-name` span，以完整名称开始，每次挂载调用一次 `observeName`），宿主用适配后的文本调用 `setText`。Plannotator 的 `observeTreeName`（随旧行删除）使用共享的 `packages/ui/utils/textFit.ts`：一个 ResizeObserver 处理所有适配名称（用其 `contentRect` 宽度，不读取布局），canvas `measureText` 按字体和文本缓存，一个 `document.fonts` 监听器，能放下的名称不做适配工作。放不下的名称变为词干开头、省略号、始终显示的尾部（词干最后三个字素加扩展名，如 `st…ep0.py`）；连 `…` 和尾部都放不下时，显示 `…` 和尾部末尾尽可能多的内容。按字素截断，不会画出半个字母。行保留完整名称作为 `aria-label` 和 title。文件夹及无 `observeName` 的行保留普通末尾省略号。早期版本在补丁内用每行隐藏探针测量，200 行时滚动造成约 1.3 秒强制布局。移除条件：若上游新增按完整字母、保留尾部的截断，将两个树视图切换过去，并从补丁中删除 `TailKeptName` 和 `observeName` |
| `onDirectoryExpand(path)`（树选项） | 所有文件列表懒加载文件夹：未加载的文件夹是路径以 `/` 结尾；打开时加载其子项并 `batch` 加入 |
| `searchText(path)`（树选项） | 搜索查询对路径匹配时使用的文本，而非路径本身。Changes 树将分组标题和合并文件夹绘制为合成路径（`Code/src∕components/x.ts`）；它返回文件真实路径、文件夹真实路径，分组标题返回 `''`，因此查询永不匹配标题标签，`src/components` 能找到合并文件夹的文件 |
| 行上无 type-to-search | Pierre 在聚焦行上输入任何字母或数字即打开搜索框并拦截按键。Fork 删除该分支（`isSearchOpenSeedKey`），使页面单键快捷键（V、A、J/K 等）在聚焦行上生效；在搜索框内输入仍会过滤 |
| `model.refreshRows()` | 宿主状态（已查看、已暂存、评论）变化后重新运行 `renderParts` 和 `renderRowDecoration` |
| 行各部分的 Shadow part：`row-spacing`、`row-icon row-icon-file|directory`、`row-lead`、`row-git row-git-file|directory`、`row-content`（+ `row-content-muted`、`row-content-<tone>`）、`row-decoration`、`row-trail`（+ `row-lead-<tone>`、`row-trail-<tone>`）、`row-part`、`row-part-<id>`、`row-part-pressed`，以及查找框上的 `search` / `search-open` | 用 `::part()` 从 shadow root 外部稳定设置样式，而非针对内部属性的 CSS |
| CSS 变量 `--trees-row-lead-columns`、`--trees-row-trail-columns`、`--trees-row-part-gap`、`--trees-row-muted-opacity` 和 `--trees-row-active`（悬停、聚焦或选中行为 1，否则 0；无悬停时始终为 1） | lead 和 trail 是网格；宿主设置列宽使 `+N` 和 `−M` 在面板中对齐。`--trees-row-active` 让部分（Git add）仅悬停时显示，从 shadow root 外部控制 |

补丁约 170 行新增代码。涉及文件（均在 `dist/` 下）：`render/FileTreeView.js`（渲染和点击）、`render/FileTree.js`（传递 `rows`、`refreshRows`）、`model/FileTreeController.js`（`onDirectoryExpand`、`searchText`）、`style.js`（网格 CSS，作为独立字符串追加）、`model/publicTypes.d.ts`、`render/FileTree.d.ts`、`index.d.ts`（类型）。Source map 未更新。

## Plannotator 目前使用的内容

自树采用 Pierre 自身外观（2026-11-17）以来，Plannotator 只传 `rows.renderParts`（trail 部分、`muted`、`tone`、`title`；`lead` 仅用于分组标题图标）、`rows.onDoubleClick`、`rows.onActivate`、`onDirectoryExpand`、`searchText` 和 `refreshRows()`，并依赖已删除的 type-to-search。可点击部分（`id`、`onPartClick`）、`gitStatusPlacement: 'start'`、`nameTruncation: 'end' | 'keep-tail'` 配合 `observeName`，以及 lead 和 trail 列变量均未使用；它们留在补丁中，因为删除会触及已用选项共享的相同行。`observeTreeName` 已随旧行删除。

## 与新上游版本重新同步

1. 在 `packages/review-editor/package.json`（以及所有其他列出它的包）中升级 `@pierre/trees`，并将 `patchedDependencies` 键和文件重命名为新版本。
2. `bun install`。若 Bun 提示补丁不再适用：`bun patch @pierre/trees`，在 Bun 打印的文件夹中手动应用旧补丁（`patch -p1 < patches/...`，修复拒绝块；改动很小且基于稳定名称如 `renderStyledRow` 和 `renderFileTreeRowContent`），然后 `bun patch --commit <folder>`。
3. 运行树的测试（`FileTree.*.test.tsx`、`FileTreeNode.controls.test.tsx`）并查看审查侧边栏。

## 向上游提交

改动与上游 TypeScript 源码一一对应（`src/render/FileTreeView.tsx`、`src/render/FileTree.ts`、`src/model/FileTreeController.ts`、`src/style.css`、`src/model/publicTypes.ts`）。上游 PR 会在那里添加相同选项，外加测试。

## Pi 及其他安装

树在构建时打包进审查和 Inbox HTML。Pi 扩展发布该构建后的 HTML，不安装 `@pierre/trees`，因此 Pi 无需任何操作。