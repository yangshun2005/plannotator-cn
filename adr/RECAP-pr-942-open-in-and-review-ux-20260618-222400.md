# Recap / Handoff — PR #942：在外部应用中打开文件 + 代码审查 UX 改进

- **PR:** https://github.com/backnotprop/plannotator/pull/942
- **分支:** `feat/openfile`（已变基到 `main` @ #936）
- **日期:** 2026-06-18
- **范围:** 46 个文件，约 +2.3k / −0.4k。除两个新增服务端端点外（Bun + Pi 镜像），纯前端改动。无新增 diff 后端。

这是一份开发者交接文档。请从上到下阅读，理解每部分*为何*存在以及*去哪里*查看。

---

## TL;DR

一个主打功能——**在外部应用中打开当前文件**——加上一批来自内部试用（dogfooding）的代码审查 UX 改进，以及**一个真实 bug 修复**（Cmd+点击代码导航曾静默失效）。语义 diff 视图也从停靠面板迁移到了侧边栏手风琴。

提交记录（从旧到新）：

```
677ad27f feat: open the current file in an external app (review + annotate)
03821638 feat(review): file-header change letters and line counts
2cea9f31 feat(review): group the diff-settings cog with the Split/Unified toggle
384a4eff feat(review): collapse/expand-all toggle in the all-files view
797e4eeb fix(review): restore Cmd+click code navigation
9a957947 feat(review): semantic diff as a resizable sidebar accordion
7f7bd6f1 docs(adr): backlog spike — git graph view exploration
65f79376 fix(annotate): inline the open-in button beside the file name
```

---

## 1. 在外部应用中打开文件（主打功能）

**功能：** 在**审查文件头**和**标注文件徽章**中各有一个分裂按钮（应用图标 + 下拉箭头）。左半部分用上次使用的应用打开当前文件；下拉箭头列出所有检测到的应用（Finder/Explorer 显示、VS Code、Cursor、Zed、Sublime、终端、Xcode 等），外加复制路径 / 复制文件 diff。选择某个应用会打开它**并**将其设为新默认——**没有设置 UI**；下拉菜单*就是*偏好设置。移植自 OpenCode 的 "Open in" 控件。

**工作原理（关键部分）：这是一个 Bun/浏览器应用，不是 Electron。** 启动发生在**服务端**：
- `GET /api/open-in/apps` → `{ available, apps[] }`。远程/无头会话中 `available=false`（UI 隐藏该控件）。`apps` 是宿主机检测到的可启动子集。
- `POST /api/open-in` `{ filePath, base?, appId? }` → 解析绝对路径，**做包含性检查**，然后启动。

按类型的启动语义（`packages/server/open-in.ts` 的 `openFileInApp`）：
- **editor** → 打开文件（`open -a "App" file` / `<bin> file`）
- **file-manager（显示）** → 显示文件（`open -R` / `explorer /select,` / `xdg-open <dir>`）
- **terminal** → 打开文件的**父目录**
- **系统默认** → `open` / `start` / `xdg-open`，不带应用

所有启动都使用 **argv 数组，绝不用 shell 字符串**（防注入），参照 `packages/server/browser.ts`。

**关键文件：**
- `packages/shared/open-in-apps.ts` — 单一来源的应用目录（id、label、kind、各平台启动字段）。由服务端 + UI 导入。
- `packages/ui/components/OpenInAppButton.tsx` — 分裂按钮 + 下拉菜单。
- `packages/ui/components/icons/AppIcon.tsx` + `icons/app/*` — 应用品牌图标，**以 data-URI 形式 base64 内联**（见注意事项：单文件 HTML）。
- `packages/ui/utils/storage.ts` — `getLastOpenInApp` / `setLastOpenInApp`（cookie；每次调用的端口是随机的）。
- `packages/server/open-in.ts` + `packages/server/review.ts` 与 `annotate.ts` 中的端点。
- **Pi 镜像：** `apps/pi-extension/server/open-in-apps.ts` + `serverReview.ts` / `serverAnnotate.ts` 中的端点（`vendor.sh` 复制共享目录）。
- 审查接线：`FileHeader.tsx`（`showLabel={!isCompact}` → 宽屏时显示应用名）。路径为仓库相对路径 → 服务端针对 VCS 根解析（`resolveAgentCwd`）；`agentCwd` 已加入 `ReviewState`。
- 标注接线：`DocBadges.tsx` + 经 `Viewer.tsx` ← `editor/App.tsx` 传递（`openInAppPath = linkedDocHook.isActive ? linkedDocHook.filepath : sourceFilePath`）。

**值得了解的决策：**
- **没有 "Default app" 菜单项**（曾尝试，后移除）——上次使用是唯一的默认机制。
- **显示标签就是 "Finder"**（按平台），不是 "Reveal in Finder"。
- **路径安全：** 审查在服务端针对 VCS 根解析（不是客户端 `base`，当 `plannotator review` 从子目录运行时 `base` 是错的）；标注针对自身目录解析绝对路径。两者都做包含性检查。
- **远程/无本地检出的 PR：** 控件隐藏（磁盘上无可解析文件）。

---

## 2. 文件头变更字母 + 行数（审查）

用**路径右侧的 `A`/`D`/`R` 字母** + `+N −M` 计数替换了旧的方框状态图标，与文件树一致。**修改的文件保持空白**——`+/−` 已表明有变更，所以只有新增/删除/重命名才加字母。单文件：`packages/review-editor/components/FileHeader.tsx`（`FileStatusLetter`、`countChanges`）。计数在组件内从 patch 派生（无 prop 传递）。

此区域还有：**Pierre 内置的单文件头已禁用**（`DiffViewer.tsx`，`disableFileHeader: true`），避免渲染两个头——我们提供自己的。

---

## 3. 设置齿轮与 Split/Unified 切换分组（审查）

diff 显示设置齿轮（`DiffOptionsPopover`）现在位于与 Split/Unified 切换**同一个胶囊**内，用分隔线隔开，打开状态镜像活动段的样式。`packages/review-editor/App.tsx`（工具栏）+ `DiffOptionsPopover.tsx`。（此提交还携带了 open-in 的小型 `agentCwd` memo 接线。）

---

## 4. 全文件视图中的折叠 / 展开全部切换（审查）

全文件面板左上角一个**嵌入式角落单元格**，一次性折叠/展开所有文件（智能：有任一打开则折叠，否则展开）。`packages/review-editor/components/AllFilesCodeView.tsx` — 遍历 CodeView 项，翻转 `item.collapsed`，`updateItem`。此提交还为全文件界面开启了 `useTokenTransformer`（见 #5）。

---

## 5. 修复：Cmd+点击代码导航 ⚠️ 不明显的 bug

**症状：** Cmd+悬停/Cmd+点击符号无反应——无下划线提示，无 References 面板。`main` 上已存在，非本分支引入。

**根因（需无头浏览器 DOM 检查才发现）：** token 级交互仅在 Pierre 为每个 token 包裹 `data-char` 属性时生效，而这仅在**token 转换器**运行时发生。渲染器通过 `shouldUseTokenTransformer(options)` 启用转换器——但组件级的 `onTokenClick`/`onTokenEnter` 选项在到达 worker **之前就被丢弃**，且**高亮在 web worker 中运行**，其渲染选项来自*独立的*初始化配置。所以 token 被高亮（颜色出现）但从未获得 `data-char` → `isTokenPointerTarget` 永不匹配 → 处理器永不触发。静默无操作。

**修复：** 在 **worker 池的**初始化选项上设置 `useTokenTransformer: true`——`packages/review-editor/workerPool.tsx`——（外加 `DiffViewer`/`AllFilesCodeView` 上的组件级选项作为主线程回退）。用 puppeteer 端到端验证：0 → 1660 个 `data-char` token，悬停提示 + `/api/code-nav/resolve` 均触发。**无需升级 Pierre**（worker 接线缺口在 1.2.8 和 1.2.10 中相同）。

如果 token 交互再次失效，这是第一个要查看的地方。

---

## 6. 语义 diff → 可调整大小的侧边栏手风琴（审查）

语义 diff 过去打开一个 **dockview 面板**。现在是一个**固定在文件树底部的可折叠手风琴**——相同的实体行（复用 `SemanticDiffRows`），相同的点击导航（`openDiffFile` + 行选择），尺寸适配侧边栏，带**垂直拖拽手柄调整大小**。

- 新增：`packages/review-editor/components/SemanticDiffAccordion.tsx`（通过 `ReviewStateContext` 自包含；获取现有 `/api/semantic-diff`）。
- `useResizablePanel` 已**泛化支持 `y` 轴**（原仅宽度）——`packages/ui/hooks/useResizablePanel.ts`。向后兼容（`axis` 默认 `'x'`，现有侧边栏/面板调整器不受影响）。
- **移除**了 dock 面板 + 按钮 + 所有接线：删除 `ReviewSemanticDiffPanel.tsx`，从 `reviewPanelTypes`/`reviewPanelComponents` 移除 `SEMANTIC_DIFF`，从 `App.tsx` 和 `ReviewStateContext` 移除 `isSemanticDiffActive` + 加载状态回调。`semanticDiffAvailable` 保留——它是手风琴的显示/隐藏门控。
- CSS：`review-editor/index.css` 中的 `.semantic-diff-accordion` 作用域将面板行压缩到侧边栏尺寸。

文件头中的每文件 `sem · N` 徽章是独立、未改动的功能。

---

## 7. 标注中的内联 open-in（修复）

变基后，#936 添加了 `folder-file` 面包屑变体（Close + 文件名）。open-in 按钮曾作为独立行堆叠在*其上方*。现在它在每个文件行（源 / 链接文档面包屑）内**内联渲染在右侧**，仅在没有文件行时回退为独立形式。`packages/ui/components/DocBadges.tsx`。

---

## 8. ADR 待办探索 — git 图视图（文档，未构建）

`adr/research/SPIKE-git-graph-view-*.md`。探索了提交图视图（点击提交 → 其文件在右侧加载）。**关键发现：** diff 查看器与来源无关（消费 `rawPatch`），所以基本可复用；净新增工作是每提交 diff 模式 + 图泳道渲染；**仅 git** 的注意事项（jj/P4）。**已列入待办**，若重启则分两阶段计划。

---

## 横切注意事项 / 陷阱

- **单文件 HTML 构建。** 审查/计划应用作为单个自包含 HTML 文件发布。大型资源若作为单独文件输出会破坏这一点，所以所有应用图标在 `AppIcon.tsx` 中**以 base64 内联**，而非作为 `.svg`/`.png` URL 导入。
- **两个服务端运行时。** 每个服务端端点必须同时存在于 **Bun 服务端**（`packages/server/`）和 **Pi** 镜像（`apps/pi-extension/server/`），共享目录通过 `apps/pi-extension/vendor.sh` 供应。open-in 端点已镜像。
- **构建顺序。** 审查编辑器 UI 改动需先 `bun run --cwd apps/review build`，**再** `bun run build:hook`（hook 复制预构建 HTML）。运行 `bun run typecheck` 做跨包检查。
- **用 cookie，不用 localStorage** 做持久化（open-in 上次使用、调整尺寸）——每次 hook 调用运行在随机端口。
- **变基。** 干净地变基到 `main`（#936，"Persist saved annotate file edits in drafts"）。仅两处琐碎的列表追加冲突（`vendor.sh`、`shared/package.json`）——保留双方。重度重叠（`editor/App.tsx`、`DocBadges`、`Viewer`）自动合并正确（已验证 open-in 接线存活）。注意：徽章移除分支**尚未在 main 上**，所以那些改动不在本 PR 中。

## 如何验证

- `bun run typecheck` 和 review→hook 构建全程通过。
- open-in 跨平台启动、应用检测、两处头部放置均实机验证（`plannotator review` / `plannotator annotate .`）。
- 代码导航修复用无头 Chrome（puppeteer）DOM 检查确认——token `data-char` 存在 + 模拟 Cmd+点击时实际 `/api/code-nav/resolve` 请求触发。

## 文件索引（从这里开始）

| 区域 | 文件 |
|------|-------|
| Open-in 目录/UI | `shared/open-in-apps.ts`, `ui/components/OpenInAppButton.tsx`, `ui/components/icons/AppIcon.tsx`, `ui/utils/storage.ts` |
| Open-in 服务端 | `server/open-in.ts`, `server/review.ts`, `server/annotate.ts`, `apps/pi-extension/server/open-in-apps.ts`（+ serverReview/serverAnnotate） |
| 审查头部/工具栏 | `review-editor/components/FileHeader.tsx`, `review-editor/components/DiffOptionsPopover.tsx`, `review-editor/App.tsx` |
| 全文件 / 代码导航 | `review-editor/components/AllFilesCodeView.tsx`, `review-editor/components/DiffViewer.tsx`, `review-editor/workerPool.tsx` |
| 语义手风琴 | `review-editor/components/SemanticDiffAccordion.tsx`, `ui/hooks/useResizablePanel.ts`, `review-editor/index.css` |
| 标注 open-in | `ui/components/DocBadges.tsx`, `ui/components/Viewer.tsx`, `editor/App.tsx` |
| 待办 | `adr/research/SPIKE-git-graph-view-*.md` |

## 后续 / 待办

- 徽章移除分支落地 `main` 后重新变基。
- Git 图视图（阶段 1：提交列表 + 每提交 diff；阶段 2：图轨道）——见探索文档。
- 构建期间推迟的 open-in 小打磨（如未使用的 `zed-dark` 图标变体、几处低严重度审查备注）——如需见 open-in 提交历史。