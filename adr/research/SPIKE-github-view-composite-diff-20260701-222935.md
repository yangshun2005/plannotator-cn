# SPIKE：“GitHub 视图”复合 diff — 可行性检查

日期：2026-07-01
状态：已对照代码验证。为 `synthesis-github-view-20260701-222935.md` 和 `specs/github-view-three-stack-20260701-222935.md` 提供输入。
原型图：`adr/research/code_review_panel_three_stack_mockup.svg`

## 问题

原型图中的三栈面板（Committed / Changes / Untracked，已查看复选框为主、暂存圆点、静默暂存按钮）能否构建为**一种新的 diff 类型**加状态分组——而不是三个同时存在的 diff？

答案：**可以。** 服务端的整个状态机假定只有一个活动补丁（`currentPatch` + 一个 `currentDiffType` + 一个指纹，见 `packages/server/review.ts:202-343`）。一个复合比较适配该状态机；三个并行比较则不行。

## 复合比较

“如果我现在推送，GitHub 会看到什么” =
merge-base(origin/default, HEAD) → 工作区，加上未跟踪文件。

确切命令，全部为已使用的原语（`packages/shared/review-core.ts`）：

```
git merge-base --end-of-options <base> HEAD          # 同 merge-base 模式，:691
git diff --no-ext-diff [-w] --src-prefix=a/ --dst-prefix=b/ --end-of-options <mb>
                                                     # <mb> 不带 ..HEAD → 与工作区比较
+ getUntrackedFileDiffs()                            # :445 — /dev/null --no-index 合成 diff
```

分区划分是附属数据，不是 diff：

```
git status --porcelain                               # 工作区状态（已为指纹运行，:846）
git diff --name-only <mb>..HEAD                      # “分支上已更改”的文件集
```

- **Committed** = 在 mb..HEAD 集合中，status 中干净
- **Changes** = 已跟踪，status 中脏（M/A 合并；暂存位来自 status 第 1 列）
- **Untracked** = `??` 行

各分区之和恰好等于复合补丁的文件列表。

## 触点清单（全部已验证，附行号引用）

服务端 / 共享 — 各新增一个 `DiffType` case：

| 文件 | 内容 | 位置 |
|---|---|---|
| `packages/shared/review-core.ts` | `DiffType` 联合类型 | :12-27 |
| " | `runGitDiff` switch — 新 case（mb 查找 + `git diff <mb>` + 未跟踪） | :548-764 |
| " | `getGitDiffFingerprint` — 新 case：HEAD sha + `hashDiffOutput([mb])` + `hashUntracked()` + base tip | :804-893；uncommitted/branch case 恰好组合了所需部分 |
| " | `getFileContentsForDiff` — 新 case：old=`gitShow(mb)`，new=`readWorkingTree` | :895-966 |
| " | `getGitContext` diffOptions 列表 — 新选项 | :388-408 |
| `packages/shared/vcs-core.ts` | `GIT_DIFF_TYPES` 集合 | :126 |
| " | `canStageFiles` — 添加新类型（工作区文件，暂存有效） | :182-185 |
| " | `resolveInitialDiffType` — 遵循 diffOptions，若选项已发出则无需更改 | :509-525 |
| `packages/shared/config.ts` | `resolveDefaultDiffType` — 将新类型加入白名单，翻转默认值（当前为 `'unstaged'`） | :227-231 |
| `packages/server/review.ts` | `/api/diff` + `/api/diff/switch` — 对 diffType 泛化，只需将分区附属数据加入载荷 | :912-955, :1000-1102 |
| `packages/server/agent-review-message.ts` | `getLocalDiffInstruction` — 新 case（target/inspect 文本） | case switch 位于 :193-257 |
| Pi 镜像 | `apps/pi-extension` 消费 review-core 的构建时副本；`serverReview.ts` 需镜像附属数据字段 | — |

客户端：

| 文件 | 内容 |
|---|---|
| `packages/review-editor/hooks/useGitAdd.ts:17` | `STAGEABLE_DIFF_TYPES` 集合 — 添加新类型 |
| `packages/review-editor/components/DiffTypePicker.tsx:18-35` | `OPTION_HINTS` — 新选项的提示 |
| `packages/review-editor/components/FileTree.tsx` | 承载所有选择器（:395-470）和扁平树渲染（:472+）。新的分区面板是同级视图，不是 FileTree 重写 |
| `packages/review-editor/App.tsx` | 载荷已泛化地传递 `diffType`/`base`；添加 `sections` 状态 + 分区/树切换 |

## 基线陈旧性（“origin/main 真的是 GitHub 吗？”）

`detectRemoteDefaultBranch`（`review-core.ts:213-234`）已在服务端启动时运行
`git ls-remote --symref origin HEAD`（在 `review.ts:353-357` 触发）
— 并且**丢弃了 SHA 行**。同一输出包含 `<sha>\tHEAD`。保留它，
与本地 `git rev-parse origin/<branch>` 比较，启动时即可免费获得陈旧性检测。定期复查可在现有 5s `/api/diff/fresh` 轮询旁挂一个更慢的节奏（这是网络调用——约 60s 节奏，而非 5s）。

## 本仓库中的先例

`SPIKE-git-graph-view-20260618-220909.md`（已积压）独立确认：
查看器与补丁来源无关（`rawPatch` → `parseDiffToFiles` → 同一流水线），扁平提交列表 + 每提交 diff 是小规模后端新增。那正是未来的第三个切换（分区 | 树 | 提交）。

## 已检查的边界情况

- **在默认分支上：** merge-base == HEAD → Committed 分区为空；Changes/Untracked ≡ 当前的未提交视图。干净降级。
- **无远程：** `getDefaultBranch` 回退到本地 `main`/`master`（:195-201）。视图仍可用；陈旧性横幅只是永不显示。
- **无 HEAD（全新仓库）：** 未提交模式已用 `rev-parse --verify HEAD` 防护（:586）；新模式需要相同防护（mb 查找失败 → 回退到仅未跟踪）。
- **文件在分支上已提交且现在被编辑：** 只出现一次，在 Changes 中；其 diff（mb → 工作区）显示完整故事。已确认决定——无每文件“自上次提交以来”切换；该意图保留在高级菜单中作为现有的 `last-commit` 类型。
- **已暂存但在工作区被还原：** `git diff <mb>` 读取工作区，因此文件从补丁中消失；status 仍显示其已暂存。仅外观问题——圆点渲染在空 diff 的文件上。可接受。
- **jj / p4 / workspace：** 超出范围。`since-base` 通过 `GIT_DIFF_TYPES` 归 git 所有；其他 VCS 类型保持其当前默认视图不变。

## 结论

未发现阻塞项。这是一个新的 git diff 模式（沿现有、成熟的接缝约 6 处小 case 新增）、一个载荷附属数据、一个新面板组件，以及一次配置默认值翻转。原型图中看起来昂贵的部分——三个分区——是服务端大多已计算数据的客户端分组。