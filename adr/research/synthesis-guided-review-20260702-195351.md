# 综合：引导式审查

日期：2026-07-02
输入：ADR 006（`adr/decisions/006-guided-review-first-class-feature-20260702-192821.md`）
来源：
- `SPIKE-guide-provider-tour-pattern-20260702-194831.md`
- `SPIKE-guide-launch-settings-reuse-20260702-194831.md`
- `SPIKE-guide-diff-annotation-reuse-20260702-194831.md`
- `SPIKE-guide-takeover-layout-20260702-194831.md`

## 结论

ADR 的范围校准成立。该功能所需的每个子系统都已存在，且都有经过验证的接缝，正好适合这类扩展。实现包括：一个新的服务端模块、在现有接缝中新增一个 provider 分支、在 App.tsx 中新增一个布局分支，以及若干组合现有组件的新组件。两处小重构，零重写。

## 各 spike 的共识

**1. tour provider 是正确的骨架——但有一处刻意偏离。**
tour 展示了完整的 provider 生命周期：自包含模块（prompt + JSON schema + 各引擎 CLI 构建器 + 解析器 + 内存会话）、`packages/server/review.ts` 中 `buildCommand` 和 `onJobComplete` 里单个 `if (provider === ...)` 分支（review.ts:690-700, 901-914）、两条结果路由（review.ts:1016-1034），以及失败即关闭的空输出处理（无效 guide → 任务 `failed`，因此 UI 永远不会打开 404）。

偏离之处：**tour 的锚点嵌入的是模型虚构的 `hunk` 文本，以只读方式渲染**（`DiffHunkPreview`，零注解接线）。这违反我们的硬约束。guide schema 必须引用真实变更集——文件路径（针对当前 `DiffFile[]` 校验）——以便各节通过现有注解流水线渲染实际补丁的实时切片。

**2. diff + 注解复用几乎零成本。**
`DiffViewer` 已与 dock 无关；所有 dock 耦合都在约 110 行的 `ReviewDiffPanel` 包装器中。guide 需要自己的同等轻量适配器，每个被引用文件一个 `DiffViewer`。`useAnnotationToolbar` 已支持多个同时实例（模块级草稿 `Map`，以 `filePath` 为键，由 `isFocused` 仲裁）。`ReviewStateProvider` 包裹整个应用主体，因此其中的 guide 界面可免费获得注解/任务/配置。`DiffFile.patch` 是自包含的单文件补丁——`state.files.find(f => f.path === path)` 就是 v1 锚点解析的全部。`AllFilesCodeView`/`CodeView` 在每节 1–5 个文件时是错误工具。

**3. 接管是布局分支，不是覆盖层，且 dock 必须用 CSS 隐藏。**
`TourDialog`/`PRSwitchOverlay` 是 `fixed inset-0` 覆盖层——不适合作为主模式。guide 是主内容行中的条件分支（App.tsx:2671+）。关键：App.tsx 已证明卸载 `<DockviewReact>` 会销毁面板/标签布局（`files.length === 0` 分支在重新挂载时从头重建）。因此接管**用 CSS 隐藏 dock 包装器**（App.tsx:2793）和文件树，而非卸载它们。这也使 diff 切换 epoch 守卫和草稿去重与 `guideOpen` 完全解耦。

**4. 启动机制复用是一个设置切片加共享原语。**
`useAgentSettings` 是普通的 cookie 支持 Hook——添加 `guideEngine`/`guideClaude`/`guideCodex`，镜像 tour 切片。引擎/模型/努力选择器是 AgentsTab 中用本地原语（`ConfigRow`、`SegmentedPicker`、`SelectMenu`）构建的内联 JSX；将原语提升到共享文件，而非提取单体启动组件——类 Notion 的接管不应继承密集的侧边栏样式。

**5. 一个 SSE 连接；按 tour 方式观察完成。**
`useAgentJobs()` 只有一个调用点（App.tsx:350）。guide 界面通过 props/context 消费该实例。完成检测复制 tour 自动打开 effect（App.tsx:780-792）：监视共享任务中 `provider === 'guide' && terminal`，通过 ref `Set` 去重。

**6. Pi 镜像分两部分，手工工作不可避免。**
`vendor.sh` 将业务逻辑模块逐字 vendor 到 `apps/pi-extension/generated/`（在那里添加 `packages/server/guide/guide-review.ts`），但 `serverReview.ts` 路由是手工维护的 `node:http`——guide 路由需手工镜像，与 tour 相同。

## 已识别的缺口 / 重构（小且有界）

1. **`isFocused` 仲裁不感知 dock 可见性**（App.tsx:1778, ReviewDiffPanel.tsx:22）。隐藏的 dock 面板仍会抢占工具栏焦点，并与 guide 的 `DiffViewer` 争抢同一文件的草稿。修复：将 dock 侧的 `isFocused` 以 `!guideOpen` 为门控（并给 guide 侧实例自己的仲裁器）。
2. **日志格式化器的 provider 检查**（`agent-jobs.ts:312-314`）硬编码了 Claude 引擎 stdout 格式化的 provider——必须添加 `guide`，否则日志会原样渲染。
3. **锚点漂移**：guide 是针对补丁快照生成的；其下的 diff 可能变化（agent 在审查中途提交——陈旧指纹正是为此存在）。文件级锚点可优雅降级（路径缺失 → 该节显示"已过时"标签而非 diff）；行范围锚点则不能。这是 v1 采用文件级的又一理由。

## ADR 006 遗留问题的解决（提议）

- **持久化**：guide 会话中的内存 `Map`（tour 先例），外加已审查状态的 PUT 端点。持久存储仍推迟；若日后以磁盘为后端，API 形态不变。
- **单个 vs 多个 guide**：guide 界面显示最新的已完成 `guide` 任务；重新生成即替换。较旧的 guide 任务仍可作为任务访问（结果按 jobId 保留），但 UI 建模为一个活动 guide。现在成本低，且不妨碍日后加选择器。
- **锚点粒度**：v1 采用文件级，schema 中为每个 diff 引用保留可选的 `lineStart`/`lineEnd`（渲染为滚动/高亮提示，而非切片）。仅当 v1 证明不足时才做 hunk 级切片——且届时从 `DiffFile.patch` 切真实 hunk，绝不合成。