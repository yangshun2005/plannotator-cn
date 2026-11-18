# 规格：默认 GitHub 视图与三栏面板

日期：2026-07-01
状态：**决策已于 2026-07-01 敲定 — 可进入 ADR**
原型图：`adr/research/code_review_panel_three_stack_mockup.svg`
研究：`adr/research/SPIKE-github-view-composite-diff-20260701-222935.md`、
`adr/research/synthesis-github-view-20260701-222935.md`

## 摘要

让默认代码审查界面零配置回答"我在 GitHub 上会看到什么"。新增一种 git diff 类型——merge-base(base, HEAD) → 工作区 + 未跟踪文件——渲染为三段式文件面板（已提交 / 变更 / 未跟踪），以"已查看"作为每行的主要操作。现有功能全部移入渐进式披露；不删除任何内容。

## 1. 新 diff 类型：`since-base`

**已决定。** 内部 id 为 `since-base`；用户可见标签动态生成：**"Since main"** / "Since origin/main" / "Since <base>"——如实反映比较对象，跟随 base 选择器，不带平台品牌。

**语义：** 用户此刻提交并推送后，GitHub PR 中会出现的全部内容，比较基准为与审查 base（默认 `origin/<default-branch>`）的 merge-base。

**计算**（`packages/shared/review-core.ts` 中 `runGitDiff` 新增分支）：

```
mb = git merge-base --end-of-options <base> HEAD
patch = git diff --no-ext-diff [-w] --src-prefix=a/ --dst-prefix=b/ --end-of-options <mb>
      + getUntrackedFileDiffs()
label = "Since <base>"
```

保护：无 HEAD（全新仓库）或 merge-base 失败 → 回退为仅未跟踪文件的 patch，并附带非致命错误标签，与 `uncommitted` 的 `hasHead` 保护模式一致。

**配套改动（各一处）：**

- `getGitDiffFingerprint`：`HEAD sha` + `hashDiffOutput([mb])` + `hashUntracked()` + base tip sha。
- `getFileContentsForDiff`：旧 = `git show <mb>:path`，新 = 工作区。
- `getGitContext` diffOptions：存在默认分支时最先输出 → 通过现有 `resolveInitialDiffType` 成为初始类型。
- `vcs-core.ts`：加入 `GIT_DIFF_TYPES`；`canStageFiles` 返回 true。
- `agent-review-message.ts` 的 `getLocalDiffInstruction`：目标 = "自与 <base> 的 merge-base 以来的所有变更（已提交和未提交），加上未跟踪文件——即 GitHub PR 会展示的完整集合"；检查 = mb 查找 + `git diff <mb>` 命令。
- `config.ts` 的 `resolveDefaultDiffType`：接受新 id；**所有人的默认值切换为它**（原为 `'unstaged'`）——已决定。配置中显式设置了 `defaultDiffType` 的用户保留其选择。
- 客户端 `STAGEABLE_DIFF_TYPES`（`useGitAdd.ts`）和 `OPTION_HINTS`（`DiffTypePicker.tsx`）：加入新 id。
- **`WORKTREE_SUB_TYPES`**（`review-core.ts:521`）：加入 `since-base`，使 `worktree:<path>:since-base` 可解析，且切换工作区时保留该模式（App.tsx 约 1200 行处镜像了此解析——需保持同步）。*（预检）*
- **`compareTarget.diffTypes`**（`getGitContext`，`review-core.ts:427`，当前为 `["branch", "merge-base"]`）：加入 `since-base`，使 base 选择器行对其开放——这正是"vs origin/main"入口的支撑。*（预检）*
- Pi：`vendor.sh` 在构建时将 `packages/shared/` **以及** `packages/server/agent-review-message.ts` 全部 vendor 到 `generated/`，因此所有共享层改动自动传播。仅需手动镜像 `sections` 载荷字段和 `serverReview.ts` 中的 `POST /api/fetch-base`。*（预检已验证）*

## 2. Sections 附属数据

服务端在生成 patch 的同时计算（`/api/diff` 和 `/api/diff/switch` 响应中，当 `diffType === "since-base"` 时）：

```ts
sections: {
  base: string;            // 解析后的 base ref，例如 "origin/main"
  mergeBase: string;       // sha，用于展示/调试
  files: Record<string, {
    group: "committed" | "changes" | "untracked";
    status: "A" | "M" | "D" | "R";   // 相对 merge-base（来自 patch 本身）
    staged: boolean;                  // porcelain 第 1 列
  }>;
}
```

数据来源：`git status --porcelain` + `git diff --name-only <mb>..HEAD`。
分区规则：`??` → 未跟踪；status 中有改动 → 变更；否则（在 mb..HEAD 集合中且工作区干净）→ 已提交。同时出现在分支集合和脏文件集合中的文件归入**变更**（其 diff 仍展示完整的 since-base 内容——已确认，不做逐文件覆盖）。

## 3. UI：三栏 Sections 面板（按原型图）

新组件（`FileTree` 的兄弟组件，同一左侧面板位置）：

- **头部：** "Review" + 分支 chip；其下方一个低调的 "vs origin/main" 入口 → 打开现有 `BaseBranchPicker`。
- **三个可折叠分区**带计数：已提交（约 55% 不透明度弱化）、变更、未跟踪。
- **已提交分区高度随视口自适应**（已决定）：变更和未跟踪始终完整渲染——每个可操作文件无需滚动即可见。已提交填充剩余垂直空间，空间紧张时至少保留几行可见，然后显示"N more files"展开行。展开状态在面板内滚动。
- **行结构**（来自原型图）：已查看复选框（主要操作，沿用现有 `viewedFiles` 行为，含 `v` 键和暂存后自动标记已查看）· 文件名 · 暂存圆点（紫色）· 低调的暂存/+ 按钮（`/api/git-add`）· 状态字母（M 琥珀 / A 绿 / U 灰；存在时 D 红；未修饰的修改文件沿用树视图惯例——sections 视图按原型图显示 M）。
- 点击行打开同一个单文件 DiffViewer；All-files 和 Semantic 面板不变（它们已消费扁平的 `DiffFile[]`）。

**视图切换：** 面板头部 `Sections | Tree`。Tree = 当前 `FileTree` 渲染同一 since-base patch。选择持久化（cookie，与其他设置一致）。预留第三槽位：`Commits`（已排期的 git-graph 研究，Phase 1）。

**渐进式披露：**

- 默认界面：不显示任何选择器。
- "vs origin/main" → base 选择器（第 1 层）。
- "Advanced" 入口（已决定：**低调的页脚行**，位于现有 copy-diff 页脚附近）→ 今天的 `DiffTypePicker` 完整模式列表（staged/unstaged/last-commit/merge-base/all/worktrees）。选择高级模式后，面板切换为 Tree 视图（sections 仅存在于 since-base）。
- `DiffTypeSetupDialog` 新增 "GitHub view (recommended)" 作为默认选项；高级用户仍可选经典默认。

## 4. 基线过期横幅

- 扩展 `detectRemoteDefaultBranch`（或新增兄弟函数），从它已解析的同一 `ls-remote` 输出中同时返回远端 tip sha。
- 与 `git rev-parse origin/<branch>` 比较。不同 → 载荷/新鲜度探测字段 `baseBehindRemote: true`。
- UI：低调横幅 "Baseline is behind GitHub · Fetch" → 专用端点 `POST /api/fetch-base` 执行 `git fetch origin <branch>`，然后客户端重新运行 `/api/diff/switch`（preserveFile）。（已决定：独立端点，不重载 `/api/diff/switch`——fetch 是网络变更操作，有其自身失败模式。）
- 频率：启动时检查 + 每约 60 秒一次（网络调用——不放在 5 秒指纹轮询中）。

## 5. 范围

- **仅 Git。** jj → 保留 `jj-current` 默认值和当前 UI；p4 和工作区模式不变。`since-base` 通过 `GIT_DIFF_TYPES` 归 git 所有。
- PR 模式不变（它本身就是 GitHub 视图）。
- 不删除任何现有 diff 类型、端点或控件。

## 6. 验收草案

1. 在含提交 + 编辑 + 新文件的功能分支上全新运行：面板显示三个分区，计数正确，每个文件 diff = since main。
2. 在默认分支上有本地编辑：已提交为空，变更/未跟踪与今天的未提交视图一致。
3. 从行按钮暂存文件：圆点出现，文件自动标记已查看，文件留在变更区。
4. 切换 Sections ↔ Tree：相同文件，不重新获取。
5. `origin/main` 过期：横幅出现；Fetch 刷新并重新分区。
6. Advanced → "Staged changes"：经典行为，Tree 视图，无 sections。
7. jj 仓库：行为不变。

## 决策日志（2026-07-01）

- [x] 标签：**"Since main"**（按 base 动态）；内部 id `since-base`。
- [x] 推出：**全员切换**；显式 `defaultDiffType` 配置予以尊重。
- [x] 已提交分区：**视口自适应**——变更 + 未跟踪始终完整可见；已提交填充剩余空间，带最小行数下限和"N more files"展开。
- [x] Advanced 入口：**页脚行**。
- [x] 过期时获取：专用 `POST /api/fetch-base`。