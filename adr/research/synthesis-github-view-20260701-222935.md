# 综合：代码审查的“GitHub 视图”默认

日期：2026-07-01
输入：`SPIKE-github-view-composite-diff-20260701-222935.md`、
`SPIKE-git-graph-view-20260618-220909.md`（已积压，此处复用）、
原型 `code_review_panel_three_stack_mockup.svg`、2026-07-01 产品讨论。

## 问题（用户原话）

- “我只想看到我在 GitHub 上会看到的东西。”
- “我只想看到我运行 git status 时会发生什么。”

当前的 `DiffTypePicker` 迫使用户在渲染任何内容之前，先把这些意图翻译成 git 机制（staged / unstaged / merge-base / all）。没有任何选项能回答 GitHub 那个问题：`merge-base`（我们自己的提示甚至写着“与 GitHub 的 PR 视图相同”）排除了工作区，而 `unstaged` 默认对刚提交过的人什么都不显示。任何可能的默认都会背叛这两种心智模型之一。

## 解决方案

不要挑更好的默认——消解这个选择。一个比较同时服务两种心智模型：

> **merge-base(origin/default, HEAD) → 工作区 + 未跟踪文件**
> = “如果我现在推送，GitHub PR 会显示的内容”

- **diff 内容**回答 GitHub 那个问题。屏幕上每个 diff 都表示同一件事：自 main 以来的变更。没有例外，没有逐文件模式。
- **分组**回答 git-status 那个问题：三个区块——已提交（弱化）、变更（修改 + 新增合并，暂存圆点）、未跟踪——正是原型的三栈面板。

讨论中确认的决定：

1. **每个 diff 都显示“自 main 以来”。** 一个文件在旧提交中改过、现在又改过，只出现一次（在“变更”中），其 diff 显示自 main 以来的全部内容。“仅自上次提交以来”不是此视图上的开关——该意图已作为 `last-commit` diff 类型存在，位于高级菜单中。
2. **Viewed 不可动。** 它是应用最常用的功能；原型把复选框提升为该行的主操作。现有 `viewedFiles` 基础设施原样沿用。
3. **Sections | Tree 切换**，未来还有第三种模式：Commits（积压的 git-graph spike 的 Phase 1——扁平提交列表 + 逐提交 diff——无需改动查看器即可接入此处）。
4. **渐进式披露**：默认屏幕零选择器。“vs origin/main”标签打开现有的 BaseBranchPicker。一个 Advanced 入口暴露今天的完整 diff 类型列表。什么都没删除；它只是不再是前门。
5. **基线诚实**：我们对比的是 `origin/main` = 上次 fetch 时的 GitHub。启动时的 `ls-remote` 调用已经收到远端 tip SHA 却把它丢弃；保留它、比较，并在过期时显示一条安静的“基线落后于 GitHub——请 fetch”横幅。正是这条横幅让“这是 GitHub 视图”成为可信的声明。
6. **仅 Git。** jj 和 Perforce 保留当前 UI。工作区模式暂时保留当前 UI。
7. **已提交区块的默认状态**（折叠 vs 窥视）：实现时感受一下；原型以 55% 不透明度弱化它，并带“还有 124 个文件”的截断，这暗示带计数的折叠是起点。

## 为什么这很便宜（spike 发现，精简版）

三栈面板是**一个新 git diff 类型加一个状态 sidecar**，不是新的 diff 引擎：

- 服务器的单快照状态机（一个 `currentPatch`、一个指纹）原样适配复合比较。
- `runGitDiff`、指纹计算、文件内容获取、暂存、新鲜度探测和 agent 上下文构建器各自只需沿现有接缝新增一个 `case`（约 6 处小改动，spike 中已逐行验证）。
- 区块划分来自 `git status --porcelain`（已为指纹运行）+ `git diff --name-only <mb>..HEAD`——作为载荷 sidecar 下发；客户端据此分组文件。
- Pierre diff 查看器零改动（已再次确认；也是 git-graph spike 的关键发现）。
- Viewed、暂存（`/api/git-add`）、注释和反馈导出均无需改动即可工作；针对 mb→worktree 的注释与今天的 merge-base 注释形状相同。

真正的新工作是 UI：Sections 面板组件、Sections/Tree 切换、披露重新分层（选择器 → Advanced）以及过期横幅。

## 带入规格的开放问题

- 新 `DiffType` id 及其用户可见标签的命名。
- 基线过期时 fetch 操作的位置（横幅按钮 → 服务端 `git fetch`，然后重新切换）。
- Sections 视图是完全取代 FileTree 作为默认，还是保留每用户偏好（cookie，与其他设置一样）。
- sections sidecar 的确切载荷形状。