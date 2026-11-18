# 更新与"新功能"生命周期状态机

`update-lifecycle.qnt` 是一个 [Quint](https://quint-lang.org) 模型，描述 Plannotator 安装如何得知发布版本、更新、重启并显示"新功能"。模型依据 main 分支代码（`35f9ad90d`）编写，覆盖单机、单数据目录。PR #2338 的描述列出了模型发现的问题，以及真实代码复现了哪些。

模型与代码通过两种方式关联：

- **脚本化运行**（spec 中的 `run …Test`）：每个发现的最小轨迹，在模型中重放。
- **fast-check 对照测试** `packages/server/update-lifecycle.model.test.ts`：用 TypeScript 实现相同的状态和规则，通过相同动作与真实代码并行驱动。每个动作后，真实按钮、真实链接和 GitHub 收到的请求必须与模型一致。

## 如何运行

```sh
# 脚本化轨迹（每个实例模块单独运行）
quint test --main onMain      spec/update-lifecycle.qnt
quint test --main onFix       spec/update-lifecycle.qnt
quint test --main onWhatsNewFix spec/update-lifecycle.qnt
quint test --main onFixDraft  spec/update-lifecycle.qnt
quint test --main onWindows   spec/update-lifecycle.qnt
quint test --main onLocks     spec/update-lifecycle.qnt

# 随机模拟：每个实例上的每个属性（约 5 分钟）
bash spec/simulate.sh
SAMPLES=50000 PROPERTIES="WhatsNewWithinRun" INSTANCES="onMain onFix" bash spec/simulate.sh

# 保留违规轨迹，并将其中一条打印为普通步骤
TRACES=/tmp/traces bash spec/simulate.sh
bun spec/show-trace.ts /tmp/traces/onMain-WhatsNewWithinRun.itf.json

# 模型与真实代码并行（fast-check）。Todo 测试即发现的问题。
bun test packages/server/update-lifecycle.model.test.ts
bun test --todo packages/server/update-lifecycle.model.test.ts   # 运行发现的问题（在 main 上失败）
PLANNOTATOR_MODEL_RUNS=500 bun test packages/server/update-lifecycle.model.test.ts
```

`simulate.sh` 使用 Quint 的 Rust 模拟器（`BACKEND=typescript` 使用较慢的版本）。首次使用时，Quint 的 Rust 后端会从 Quint 的 GitHub 发布版本下载求值器到 `~/.quint`。此处不运行 `quint verify`，那需要 Apalache，本次工作未下载。

## 实例

| 模块 | 对应的代码 |
|---|---|
| `onMain` | `35f9ad90d` 时的 main |
| `onFix` | PR #2337：发布不足 2 天时，空的"新功能"回答在 15 分钟后重新询问，而非 24 小时 |
| `onWhatsNewFix` | #2337 加上 F2-F4 修复（`WHATS_NEW_FIX`）：无人应答的笔记请求在 15 分钟后重新询问；可能是网络原因导致的失败（失败后无发布检查应答）在发布检查应答后 15 分钟重新询问一次；`whatsNewPending` 在 Inbox 启动间保留"新功能"，直到以可读笔记提供过 |
| `onFixDraft` | #2337 加上其 release.yml 变更：草稿发布时已包含写好的笔记 |
| `onWindows` | main，采用 install.ps1 的替换方式（先移开再移入）和 Windows 锁（包装器的 pid，不持有超过一小时） |
| `onLocks` | main（仅脚本化锁轨迹：F5 和 F6 已修复） |

每个实例的锁规则均来自 PR #2342：`launchInstaller` 在生成时以包装器名义写入锁；安装器仍在运行的锁在超过一小时后最多持有一天（POSIX）；不再持有的锁被释放并记录为失败；包装器只释放自己的锁。

## 属性说明

每个属性是 `quint run` 在每个状态检查的不变量。"最终"属性使用边界：自其前置条件开始经过边界后，结果必须成立。

| 属性 | 含义 |
|---|---|
| `NoTwoInstallers` | 两个安装不会同时运行，除非第一个已超过锁的限制（POSIX 为一天，Windows 为一小时）。 |
| `BinaryAlwaysThere` | 任何步骤的终止都不会使安装路径缺少完整二进制。 |
| `InboxCanStart` | Inbox 启动（或重启）总能找到可运行的二进制。 |
| `NothingFetchedWhileChecksOff` | `PLANNOTATOR_UPDATE_CHECK` 关闭时，无自动请求访问 GitHub。立即检查和立即更新是用户自己的请求，允许。 |
| `RequestRateBounded` | 无请求循环。自动发布检查至少间隔 1 分钟。同一发布笔记的请求至少间隔 15 分钟。 |
| `StaleLockRecovered` | 无安装存活时的"更新中…"不会持续一小时。 |
| `PersonCanUpdate` | "更新中…"隐藏立即更新和重试，因此仅在安装真正运行时显示。 |
| `WhatsNewWithinRun` | **0.28.37 缺陷。** 更新后，在新版本上启动的 Inbox 运行中，"新功能"链接在笔记写入后 30 分钟内显示。前置条件排除了另外两个原因：网络从未失败，且无笔记请求被放弃。 |
| `WhatsNewAfterAbandonedRequest` | 同上，当 Inbox 在笔记请求进行中退出时。 |
| `WhatsNewAfterBlip` | 同上，网络中断后恢复时。在 `onWhatsNewFix` 上，任意时长的一次中断后成立。若网络在快速重试前再次断开（已恢复、发布检查已应答、然后再次断开），仍等待 6 小时，即当前的退避：轨迹 `flappingNetworkWaitsTest`。 |
| `WhatsNewAfterUpdate` | 更新后，链接至少显示一次，即使 Inbox 在笔记可读前重启。 |
| `NewReleaseOffered` | 较新发布在 10 分钟加一个 tick 内提供。 |
| `NewReleaseOfferedAfterAbandonedCheck` | 同上，从进程在请求中途放弃的发布检查算起。 |
| `FailedUpdateShown` | 用户启动的失败安装显示为失败，带日志和重试。排除：与用户安装并行、超过锁的一天且在之后结束的旧安装，用自己的版本覆盖了 `update-result.json`。 |

环境假设：CI 在发布后一小时内写入发布笔记（`NOTES_LAG`）。请求在时间推进前应答或超时。生成的包装器在时间推进前写入锁行（其第一条命令），或先被终止。
真实 Inbox 每分钟 tick；`advance` 跳跃 1 到 1440 分钟，并以一次 tick 结束。

## 状态与代码对应

| 模型（`Sys` 字段或定义） | 代码（main，`35f9ad90d`） |
|---|---|
| `published`、`body`、`pubAt` | GitHub：`releases/latest` 重定向（`fetchLatestFromPage`，`packages/shared/update-check.ts:700`），`releases/tags/vX`（`fetchReleaseByTag`，`:787`） |
| `latest`、`latestAt` | `update-state.json` 的 `latestRelease`（`update-check.ts:151`） |
| `rcAt`、`rcFailed`、`checkDue`、`releaseCheck` | `releaseCheck`（`:158`），`isReleaseCheckDue`（`:455`），`runReleaseCheck`（`:825`） |
| `notes` | `update-notes/<v>.json`，`writeReleaseNotes`（`:542`），"空" = `!hasReleaseNotesContent`（`:573`） |
| `nf`、`notesDue`、`notesClaim`、`notesAnswered` | `notesFetch`（`:223`），`isNotesFetchDue`（`:1053`），`fetchReleaseNotesFor`（`:1082`，认领 `:1101`，失败 `:1113`，保存 `:1121`） |
| `nf[v].inFlight`、`nf[v].quick` | `notesFetch[v].inFlight`、`notesFetch[v].retryOnline`（`WHATS_NEW_FIX`） |
| `inboxVer` | `inboxVersion`（`:211`） |
| `wnPending` | `whatsNewPending`（`WHATS_NEW_FIX`） |
| `pendingV`、`pendingPerson` | `pending`（`:199`） |
| `resultV`、`resultCode` | `update-result.json`，`readAutoUpdateResult`（`packages/server/auto-update.ts:309`） |
| `lockHeld`、`lockKind`、`lockAt`、`lockOwner` | `update.lock`：`launchInstaller` 在生成时写入的持有者行，然后是 POSIX 包装器自己的（`POSIX_WRAPPER`） |
| `lockExpired`、`holderGone`、`liveLock`、`lockHeldView`、`canAcquire`、`reap` | `lockExpired`、`lockHolderGone` / `judgeLockHolder`、`lockHeld`（`isUpdateLockHeld`）、`acquireUpdateLock`、`reapDeadUpdateLock`（`auto-update.ts`） |
| `autoLast`、`cliRun`、`autoAfterClaim` | `lastCheckAt`、`scheduleAutoUpdateCheck`（`:1077`）、`runAutoUpdateCheck`（`:547`） |
| `launch` | `beginManualUpdate`（`:698`）、`startManualUpdate`（`:742`）、`runAutoUpdateCheck` 中的启动（`:629-655`）、`launchInstaller`（`:970`） |
| `inst[i].phase` | 分离的包装器和安装脚本：锁行（`:792`）、`update-prepare` 下载（`:816`）、替换（`scripts/install.sh:1088-1099`；Windows `scripts/install.ps1:741-766`、`scripts/install.cmd:956-966`）、`record` 和 `release_lock`（仅自己的锁）、信号陷阱（`:806-814`） |
| `binOk`、`binV` | `~/.local/bin/plannotator`（`%LOCALAPPDATA%\plannotator\plannotator.exe`） |
| `ibAlive`、`ibRuns`、`startInbox` | `createInboxUpdates`（`packages/server/inbox-updates.ts:274`）、`start()`（`:584`） |
| `ibUpdated` | `updated`（`:291`），启动时设置（`:296-313`） |
| `ibOnDisk` | `onDisk`、`probeBinary`（`:464`） |
| `ibInflight` | `notesInFlight`、`ensureNotes`（`:341`） |
| `fetchable` | `fetchableVersion`（`:334`） |
| `tick` | `tick`（`:488`）；页面加载：`nudge`（`:598`） |
| `link` | `computeUpdatedLink`（`:385`） |
| `uiState` | `deriveInboxUpdate`（`:230`）经由 `compute`（`:418`） |
| `doUpdateNow` / `doCliUpdateNow` | `updateNow`（`:608`）；`POST /api/update`（`auto-update.ts:1243`）和 `plannotator update` |
| `doRestart` | `POST /api/inbox/restart`（`packages/server/inbox.ts:2608`）、`onRestartRequested`（`apps/hook/server/inbox-command.ts:302`） |

未建模：API 回退和页面速率限制（速率限制仅延长边界）、稍后提醒、发布图片、不可写状态文件、时钟回拨，以及进程间对 `update-state.json` 的读-改-写竞争。PR #2338 描述将该竞争列为代码阅读疑点。

# Claude Code 会话的 Inbox 消息落点状态机

`inbox-project.qnt` 建模一个 Claude Code 会话和一个 Inbox，同时 agent 在文件夹间移动：shell `cd`、Claude Code 自身的 `/cd`、mod 的桥接轮询、带与不带 `project_path` 的发送、用户的"新消息"和线程列表。文件夹形状类似暴露该缺陷的工作区：一个普通文件夹（无 `.git`）含普通子文件夹、一个仓库及其子文件夹、该仓库在别处的链接工作树、另一个仓库和一个临时文件夹。

```sh
quint test --main onMain         spec/inbox-project.qnt   # 缺陷轨迹
quint test --main onFix          spec/inbox-project.qnt
quint test --main onFixStartOnly spec/inbox-project.qnt
bash spec/simulate-inbox-project.sh                       # 每个属性、每个实例（秒级）
bun test packages/server/inbox-project.model.test.ts      # 模型与真实代码并行（fast-check）
```

| 模块 | 对应的代码 |
|---|---|
| `onMain` | 修复前的 main：mod 在 `$.session.cwd()` 下归档；`read_thread` 的列表会创建项目 |
| `onFix` | 修复：mod 在 `$.session.root()`（`InboxLink.projectFolder`）下归档，仅 `/cd` 会移动 |
| `onFixStartOnly` | 关闭 `INBOX_PROJECT_FOLLOWS_SLASH_CD` 的修复：启动文件夹，即使 `/cd` 后也不变 |

| 属性 | 含义 | `onMain` | `onFix`、`onFixStartOnly` |
|---|---|---|---|
| `OneProjectPerSession` | 会话的发送和新消息落入一个项目（其文件夹的），除非 agent 指定 `project_path`。shell `cd` 从不改变它。 | 违反 | 成立 |
| `NewMessageAgreesWithSend` | 新消息落在该时刻会话自身发送会落的位置。 | 违反 | 成立 |
| `NoProjectFromCd` | 每个项目都是启动文件夹的、`/cd` 目标的或 agent 指定的。 | 违反 | 成立 |
| `AttachmentsWhereAgentWorks` | 相对附件基于当前工作文件夹解析；指定的 `project_path` 保持相对它。 | 成立 | 成立 |
| `ReadMakesNoProject` | 线程列表从不创建项目。 | 违反 | 成立 |
| `NamedProjectHonoured` | 指定的 `project_path` 即为项目。 | 成立 | 成立 |

已知窗口：在 `/cd` 与下次轮询之间的一秒内发送的新消息落在会话原位置（轮询才告知 Inbox）。关于新消息的两个属性针对自上次 `/cd` 后已轮询的会话陈述。

fast-check 对照测试（`packages/server/inbox-project.model.test.ts`）驱动 mod 的真实 `InboxLink`（工具调用和桥接轮询）和真实 Inbox 在真实文件夹和仓库上运行，并在每个动作后检查 Inbox 的项目、每条消息的项目、其附件文件和 `base_path` 与模型一致。将 mod 的项目文件夹改回 cwd，立即失败（`cd top/repo/src`、`send`）；Inbox 忽略工作文件夹时，消息的 `base_path` 失败。