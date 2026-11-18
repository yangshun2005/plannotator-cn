# 重构：提交视图结构清理（合并前，PR #994）

状态：待评审的规格说明。保持行为不变——不打算做任何功能改动。
起因是对 `feat/commit-list` 的代码质量审计；在合并前于同一分支交付。

## 原因

连续三轮评审都在同一处接缝发现真实缺陷：提交会话的状态机（列表缓存、后台轮询、HEAD 自动选中、加载遮罩）分散在 `useCommitLog.ts` 和 `App.tsx` 之间，每个 bug 都是跨文件不变量失步——加载标志卡住、自动选中过期、轮询/采纳竞态。每次修复都是再跨边界多传一个不变量。接缝本身是错的；本次把整个状态机移入一个模块，让不变量可在本地校验。

另有两项较小改动一并处理：`review-core.ts`（1,637 行）吸收了它本不该拥有的约 230 行提交栏代码块；以及一段 10 行的判断逻辑在两个服务端运行时中被复制粘贴。

## R1 — `useCommitsView`：提交会话状态机的唯一归属

将 `packages/review-editor/hooks/useCommitLog.ts` 重命名为 `useCommitsView.ts`（git mv），并把以下内容原样移入（语义逐字不变）：

| 部分 | 来源 | 说明 |
|---|---|---|
| HEAD 自动选中 effect + `commitsAutoSelectDone` ref | App.tsx | 输入：`enabled`、`activeCommitSha`、`isLoadingDiff`、内部 `commits`；调用新的 `onOpenCommit(sha)` 选项 |
| `commitsVeilActive` 判断 | App.tsx | 变为返回的 `veilActive`；输入：`enabled`、`diffError`、`activeCommitSha`、`isLoadingDiff`（选项）+ `error`/`isLoading`/`commits.length`（内部） |

新选项：`{ enabled, contextKey, activeCommitSha, isLoadingDiff, diffError, onOpenCommit }`。新返回：现有日志接口 + `veilActive`。

**有意留在 App.tsx**（各自守护 App 拥有的流程）：

- `commitsCapable` / `showCommitsPanel` 推导——无论如何都需要在全局键盘处理器（Cmd+F 守卫）之上。
- `handleSelectCommit`（用户点击）：拼接 worktree 前缀并调用 `fetchDiffSwitch`/`openAllFilesPanel`——diff 切换是 App 的领域。它也正是 hook 收到的 `onOpenCommit`，因此自动选中与用户选中在构造上共享同一条代码路径。
- Cmd+F 守卫、worktree 切换的提交回退、过期刷新的 `commitLog.refresh()` 调用、进入时清除搜索。
- `CommitsPanel` 的渲染接线和遮罩覆盖层 JSX。

不做逻辑修改——仅搬移。自动选中/遮罩表达式逐字符搬移，唯一区别是直接读取内部状态，而非通过 hook 的返回值。

## R2 — `packages/shared/commit-history.ts`：把提交栏代码块从 review-core 中抽出

移出（一条内聚的消费路径——`/api/commits` + commitInfo sidecar；均不参与 diff 类型分发）：

- `CommitListEntry`、`CommitHistoryPage`、`listCommitHistory`、`COMMIT_HISTORY_LIMIT_DEFAULT/MAX`
- `CommitDiffInfo`、`getCommitDiffInfo`

**有意留在 review-core**（与 diff 类型分发交织）：`parseCommitDiffType`、`BARE_HEX_SHA_RE`、`COMMIT_FIELD_SEP`、`splitCommitFormatFields`、`runGitDiff`/fingerprint/file-contents 的提交分支、`getEmptyTreeSha`，以及 `listRecentCommits`（既有，供 `getGitContext` 使用）。

接线：

- review-core 导出 `COMMIT_FIELD_SEP`、`splitCommitFormatFields`、`BARE_HEX_SHA_RE`（当前为私有；`listRecentCommits` 仍在使用，故不能简单移走）。
- `packages/shared/package.json` 的 exports += `./commit-history`。
- `apps/pi-extension/vendor.sh` 列表 += `commit-history`（vendor 文件以相对路径导入同级文件——与 pr-provider → pr-github 相同）。
- 更新导入：`packages/server/review.ts`、Pi 的 `serverReview.ts`（`../generated/commit-history.js`）、`packages/shared/types.ts` 的再导出。
- 测试：`listCommitHistory` + `getCommitDiffInfo` 测试套件移至新的 `commit-history.test.ts`，自带一份小型 git 测试脚手架副本（按文件划分脚手架是本仓库既有的测试风格）。提交-DIFF 测试（runGitDiff/fingerprint/file-contents）留在 `review-core.test.ts`。

## R3 — 共享的 `isSameCwdCommitSwitch(previous, next)` 判断

在 review-core 中于其输入旁（`parseWorktreeDiffType`、`parseCommitDiffType`）新增导出；两个运行时的 `/api/diff/switch` 处理器用一次调用替换各自的约 10 行内联代码。单元测试：普通提交切换、同一 worktree、跨 worktree、下一类型非提交。

## 明确不在范围内

- 任何行为改动，无论多小。
- Bun/Pi 端点重复（已记录的双运行时架构）。
- `AllFilesCodeView` 的 leadingContent 机制（滚动器拥有视口；层级正确）。
- `hashString`/`hashFingerprintPart` 重复（既有；导入会把 node 依赖拖进浏览器包）。
- R1 之外的 App.tsx 拆分。

## 执行顺序与关卡

R3 → R2 → R1（最小/机械的先做，对不变量敏感的搬移最后），各一个提交。每个提交都需通过：`bun run typecheck`（会运行 vendor.sh）、`bun test`（基线 1,857，加上新的 R3 用例）、review+hook 构建、二进制重编译。R2 之后：通过编译后的二进制对 `/api/commits` + 提交切换做实时冒烟测试。R1 之后：手动 UI 走查（进入 Commits、自动打开、点击浏览、离开/重新进入、错误路径）——客户端 hook 没有测试基础设施；接受此点。

## 风险

R3 无；R2 机械（导入/vendor 接线是唯一失败模式，由 typecheck 捕获）；R1 是需谨慎对待的一项——通过逐字搬移表达式、以及随代码一同搬移的注释中六轮评审记录的不变量来缓解。