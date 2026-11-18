# 005. 以“Since main”复合差异作为默认代码审查视图

日期：2026-07-01

## 状态

已接受

## 背景

关于代码审查应用，最常见的两条用户反馈是“我只想看到我在 GitHub 上会看到的内容”和“我只想看到运行 git status 时会发生什么”。当前默认界面两者都满足不了：DiffTypePicker 迫使用户先把意图翻译成 git 机制（staged / unstaged / merge-base / all）才能渲染出内容，而且没有任何一个选项符合 GitHub 的预期——`merge-base` 不包含工作区，而默认的 `unstaged` 对刚提交完的用户什么都不显示。任何可能的默认值都会辜负这两种心态中的一种。

一个原型（`adr/research/code_review_panel_three_stack_mockup.svg`）展示了期望的形态：一个面板，三个堆叠分区——Committed（弱化显示）、Changes、Untracked——每行的主操作是 viewed 复选框，外加一个 staged 圆点和一个低调的 stage 按钮。

一次验证性 spike（`adr/research/SPIKE-github-view-composite-diff-20260701-222935.md`）确认该原型不需要三个同时存在的差异。它是一个复合比较——merge-base(base, HEAD) → 工作区，外加未跟踪文件——这正是“如果我现在推送，GitHub PR 会显示的内容”，并且契合服务器现有的单快照状态机。分区划分是一个 sidecar，由 `git status --porcelain` 加上 `git diff --name-only <mb>..HEAD` 计算得出。每个集成点都是沿现有接缝新增的一个 case（diff runner、fingerprint、文件内容、staging 门控、agent 上下文、配置默认值）。Pierre diff 查看器无需改动。

完整理由：`adr/research/synthesis-github-view-20260701-222935.md`。
完整设计：`adr/specs/github-view-three-stack-20260701-222935.md`。

## 决策

1. **新增 git diff 类型 `since-base`**，用户可见标签为“Since main”（按 base 动态变化）：`git diff <merge-base(base, HEAD)>` 对比工作区，外加未跟踪文件。此界面上的每个差异都表示“自 main 以来的变更”——没有按文件的模式。“仅自上次提交以来”仍是高级菜单中现有的 `last-commit` 类型，而不是此视图上的开关。
2. **`since-base` 成为所有人的默认 diff 类型**（此前为 `unstaged`）。在配置中显式设置了 `defaultDiffType` 的用户保留其选择。
3. **新增 Sections 面板**作为默认左侧面板视图，按原型：Committed / Changes / Untracked，由服务器计算的状态 sidecar 分组。Viewed 复选框仍是行的主操作；stage 按钮和 staged 圆点使用现有的 `/api/git-add` 路径。尺寸自适应视口：Changes 和 Untracked 始终完整渲染；Committed 填充剩余空间，设有最小行数下限和“N more files”展开。
4. **面板头部提供 Sections | Tree 切换**（按用户持久化）；第三种 Commits 模式预留给积压中的 git-graph 工作。
5. **渐进式披露**：默认界面没有任何选择器。“vs origin/main”标签打开现有的 base 选择器。一个低调的页脚“Advanced”行暴露当前完整的 diff 类型列表。没有移除任何东西。
6. **基线过期横幅**：复用启动时 `ls-remote` 调用已返回（目前被丢弃）的远程 tip SHA，检测 `origin/<default>` 是否落后于 GitHub；显示“Baseline is behind GitHub · Fetch”，由专用的 `POST /api/fetch-base` 端点支撑。
7. **仅限 Git。** jj、Perforce、工作区模式和 PR 模式不变。

## 后果

- 两种用户心态由一个界面满足，无需配置：差异内容回答“GitHub 会显示什么”，分区分组回答“git status 说了什么”。
- 所有现有用户的默认体验都会改变；旧行为仍可通过一次页脚点击获得，显式配置的默认值会被尊重。
- `packages/shared/review-core.ts`、`vcs-core.ts`、`config.ts`、`packages/server/review.ts` 和 `agent-review-message.ts` 各自新增一个 `since-base` case；`/api/diff` 和 `/api/diff/switch` 的载荷新增 `sections` sidecar；Pi 服务器同步镜像。
- 客户端新增 Sections 面板组件、Sections/Tree 切换、页脚 Advanced 入口、过期横幅，以及 `STAGEABLE_DIFF_TYPES` 和 `OPTION_HINTS` 中的 `since-base` 条目。
- 标注、反馈导出、viewed 同步、新鲜度轮询和 diff 查看器本身不变——它们已基于单个补丁和扁平文件列表运作。
- 高级模式仅在 Tree 视图中渲染；分区仅存在于 `since-base`。