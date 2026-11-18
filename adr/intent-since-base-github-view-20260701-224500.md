# 意图："Since main" 默认视图与三栏面板

构建全新的默认代码审查体验，零配置回答"我在 GitHub 上会看到什么"：新增 `since-base` git diff 类型——将 merge-base(base, HEAD) 与工作区比较，外加未跟踪文件——并设为所有人的默认值，以三栏文件面板（已提交 / 变更 / 未跟踪）替代扁平树。

原因：用户最常见的两个抱怨是"我只想看到 GitHub 会给我看什么"和"我只想看到 git status 给我看什么"，而当前 DiffTypePicker 的默认值两者都答不了——它迫使用户先把意图翻译成 git 机制，然后才有东西渲染。复合 diff 消解了这个选择：diff 内容恰好就是未来的 PR（屏幕上每个 diff 都意味着"since main"，无一例外），而分栏分组就是 git-status 的答案。Viewed——应用最常用的功能——仍是每行的主操作。

服务端做法：在 `runGitDiff` 中新增 `since-base` 分支（先 `git merge-base <base> HEAD`，再 `git diff <mb>` 不带右侧 ref，复用现有未跟踪文件追加逻辑，保留新仓库的 hasHead 守卫），并在 `getGitDiffFingerprint`（HEAD sha + 哈希后的 mb-diff + 未跟踪哈希 + base 顶端）、`getFileContentsForDiff`（旧 = `show <mb>:path`，新 = 工作区）、`getGitContext` 的 diffOptions（最先输出，标签 "Since <base>"）、`GIT_DIFF_TYPES`、`canStageFiles`、`WORKTREE_SUB_TYPES`、`compareTarget.diffTypes`、`getLocalDiffInstruction` 中添加对应分支。计算 `sections` 附属数据（`git status --porcelain` + `git diff --name-only <mb>..HEAD`，按已提交/变更/未跟踪分区并带暂存位），当类型为 `since-base` 时从 `/api/diff` 和 `/api/diff/switch` 返回。扩展启动时的 `ls-remote` 路径，保留当前被丢弃的远端顶端 SHA，暴露 `baseBehindRemote` 标志，并新增 `POST /api/fetch-base`（执行 `git fetch origin <branch>`）。将 `resolveDefaultDiffType` 的兜底翻转为 `since-base`（显式配置仍生效，保留旧 `branch`→`merge-base` 映射）。Pi 通过 vendor.sh 获取所有共享改动；仅在 `serverReview.ts` 中手工镜像 sections 字段和 fetch-base 端点。

客户端做法：新增 Sections 面板组件，复用 FileTree 的插槽，三栏按视口自适应（变更和未跟踪始终完整渲染，已提交填充剩余空间并设最小行数下限和"N more files"展开），每行按设计稿携带 viewed 复选框 / 暂存点 / 静默暂存按钮 / 状态字母；面板头部有持久化的 Sections|Tree 切换（Commits 插槽预留）；默认屏幕零选择器，"vs origin/main" 打开现有 BaseBranchPicker，底部一行安静的 "Advanced" 暴露今天的 DiffTypePicker（高级模式仅渲染 Tree）；"Baseline is behind GitHub · Fetch" 横幅接入 fetch-base；将 `since-base` 加入 `STAGEABLE_DIFF_TYPES`、`OPTION_HINTS`、App.tsx 的 worktree 子类型解析，以及 `DiffTypeSetupDialog` 作为推荐选项。仅限 Git——jj、Perforce、workspace 和 PR 模式不变；不删除任何现有内容。

参考：
- adr/research/code_review_panel_three_stack_mockup.svg
- adr/research/SPIKE-github-view-composite-diff-20260701-222935.md
- adr/research/synthesis-github-view-20260701-222935.md
- adr/research/SPIKE-git-graph-view-20260618-220909.md（已积压；预留 Commits 切换）
- adr/specs/github-view-three-stack-20260701-222935.md
- adr/decisions/005-since-base-github-view-default-20260701-223706.md