# 回顾 — "Since main" git-status 审查视图

日期：2026-07-02
关联：`adr/decisions/005-since-base-github-view-default-20260701-223706.md`

## 已交付内容

一个复合的 `since-base` 差异 — `merge-base(base, HEAD)` 对比工作树加未跟踪文件 — 成为**默认**代码审查视图，渲染为三段式 git-status 面板（Committed / Changes / Untracked），带 `Sections | Tree` 切换、首次运行设置选择器，以及"基线落后于 GitHub"的拉取横幅。仅支持 Git；Perforce/jj/workspace/PR 模式未改动。共享逻辑位于 `packages/shared/review-core.ts`，并在两个服务端运行时中镜像（Bun `packages/server/review.ts`、Pi `apps/pi-extension/server/serverReview.ts`）。

## 实现与规范/ADR 的偏离之处

以下是实现阶段有意的调整 — 记录下来，以免在存在差异处将 ADR/规范误读为事实基准：

1. **Sections 附属数据不携带逐文件 `status`。** 规范中的 `files: Record<path, { group, status, staged }>` 实际交付为仅 `{ group, staged }` — 客户端已从补丁本身推导出 A/M/D/R。
2. **Sections 面板中没有"Advanced"页脚行。** 高级差异模式位于 Tree 视图的 `DiffTypePicker` 下拉菜单中；头部的 `Sections | Tree` 切换是两者之间的路径。Sections 面板唯一的比较控件是基线行。
3. **`DiffTypeSetupDialog` 被移除，而非扩展。** 由更丰富的两页 `ReviewSetupDialog` 替代（面板视图 + 差异类型选择器，带截图预览），从新增的"Set up review view"头部菜单项进入。
4. **Committed 区块不做透明度弱化。** 它以完整权重渲染，仅通过区块标题/分组来区分。

## 合并后审查轮次（2026-07-02）

一次七智能体审查加上应用自身的 PR 审查，暴露了一批修复，均已落地到本分支：

- Sections 重命名解析器在带引号的 token 上按 ` -> ` 拆分 → 一个脏文件可能显示为 Committed。现改为引号感知（`splitPorcelainRename`，有单元测试）。
- `/api/diff/switch` 的 epoch 在 `await req.json()` 之后捕获 → 一个慢速体的旧请求可能覆盖较新的请求。现改为在任何 await 之前捕获 epoch；`hideWhitespace` 仅在胜出时提交。两个运行时均已修复。
- 无法解析的基线不再自动降级为 since-base — 默认回退到 `uncommitted`。指纹/文件内容降级到 HEAD，以与 diff/sections 同类保持一致。
- `rm --cached` 不再为同一路径产生两个差异条目（已在跟踪补丁中的未跟踪文件被丢弃）— 修复了 Sections 面板中打开错误文件及 j/k 导航循环的问题。同时修复了潜在的 uncommitted/unstaged 情形。
- 设置中的"Default Diff View"列表现保留 sections⟺since-base 的耦合关系。
- 横幅仅将 `origin/*` 视为可拉取；裸本地基线（"main"）在启动时升级为其跟踪引用，以便 Fetch 能清除它。
- `hashUntracked` 针对仓库顶层目录（原为 cwd）解析未跟踪路径，使从子目录启动的审查不会对未跟踪编辑视而不见。
- 智能体审查指令现告知智能体枚举/检查未跟踪文件。
- 术语统一：一个"Committed changes"标签；实时的"Since <base>"标签是动态的（与头部一致），"Since main"仅保留为产品文案。

## 已知限制

- **陈旧横幅仅覆盖远程默认基线。** "Baseline behind GitHub"回答的是"`origin/<default>` 是否领先于你正在比较的对象"。如果审查者通过基线选择器选择了*不同的*基线分支，该基线不会显示陈旧警告 — 支持任意基线的上游陈旧性是独立功能，此处未构建。常见情形已覆盖：默认基线，以及裸本地默认名（`main`），它在启动时及每次 `resolveReviewBase` 调用时都被规范化为其跟踪引用。
- **已提交的删除 + 未跟踪的重建显示为普通新文件。** 如果分支提交了 `f` 的删除，而用户后来以未跟踪方式重建了 `f`，since-base 去重（`removeTrackedDeletions`）会丢弃已提交的删除，审查仅显示未跟踪的新增 — 基线版本的移除被隐藏。修复它需要两个同路径差异条目，而按路径键控的 UI（停靠面板、导航、sections 映射、已查看状态）无法表示。已接受：审查者仍能看到该文件的完整当前内容。
- **仅索引变更不是独立层。** Since-base 将工作树与 merge-base 做差异 — "如果你现在执行 `git add -A && git commit` 会落地什么"，这是产品承诺。仅存在于索引中的变更（编辑 → `git add` → 将工作树恢复为基线内容）不会出现，因为现在提交所有内容不会包含它。这是有意的语义，不是疏忽。
- **字面包含 ` -> ` 的文件名可能在侧边栏中分组错误。** Porcelain v1 不对普通空格加引号，因此涉及此类名称的重命名会在错误的分隔符处拆分，两侧都回退到 Committed 组。只有 `--porcelain -z` 能完全消歧；为一个病态名称重写解析器不值得。仅影响显示。

## 参考文件

- 决策：`adr/decisions/005-since-base-github-view-default-20260701-223706.md`
- 规范：`adr/specs/github-view-three-stack-20260701-222935.md`
- 意图：`adr/intent-since-base-github-view-20260701-224500.md`
- 研究：`adr/research/SPIKE-github-view-composite-diff-20260701-222935.md`、
  `adr/research/synthesis-github-view-20260701-222935.md`