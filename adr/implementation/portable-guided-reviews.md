# 便携式引导评审 — 实现规范

状态：已在 `portable-guides` 分支实现（见 §11）；首次 Cloudflare 部署待进行
决策记录：`adr/decisions/007-portable-guided-reviews-20260815.md`（全文引用 D1–D12）
抢救来源：分支 `guided-review-html-export`（`2cf18cc3`），位于本地 worktree

## 1. 目标

将一份引导评审分享为单个小型 HTML 文件，渲染效果与应用内引导完全一致，渲染器由 `guides.show` 提供。导出实现为纯函数，可从 Plannotator UI 和 CLI 调用。为共享反馈和托管引导奠定基础（格式、包、领域），但不构建它们。

撰写时的非目标（D11、D12）：分享链接 / 上传、导出中的标注或 PR 讨论串、额外调色板、独立 agent 生成器、商业平台的嵌入 API。分享链接后来在各自的契约 `adr/implementation/guide-share-hosting.md` 下构建（见 §11 的托管阶段）；其余仍不在范围内。

## 2. 架构

```
┌──────────────────────── 生产者 ────────────────────────┐
│ Plannotator 服务器 (Bun + Pi)          plannotator CLI     │
│  guide 任务启动 → 捕获补丁      guide export        │
│  guide-store: {id}.json + {id}.patch    --id | --snapshot  │
│  GET /api/guide/:id/export  ───────┐         │             │
└────────────────────────────────────┼─────────┼─────────────┘
                                     ▼         ▼
                     ┌── @plannotator/core/guide-format ──┐
                     │ 快照 schema v1 · 严格解析器  │
                     │ toPortableHtml(snapshot) · fixtures │
                     └────────────────┬────────────────────┘
                                      ▼  单个 HTML 文件（大小 ≈ diff）
        ┌────────────────────── guides.show (Cloudflare Worker) ──────────────────────┐
        │ /v1/viewer.<hash>.js  /v1/viewer.<hash>.css  /v1/fonts/*  /v1/langs/*.js   │
        │ 不可变、内容哈希 · /g/<id> 预留 · 独立部署，非 paste-service│
        └────────────────────────────────────┬────────────────────────────────────────┘
                                             ▼
                     ┌── @plannotator/guide-viewer ────────────────────────┐
                     │ GuideView → GuideSectionCard → GuideFileCard →      │
                     │ GuideViewportManager → AllFilesCodeView (readOnly)  │
                     │ 由 packages/review-editor 导入（应用内）         │
                     │ 打包为 viewer.<hash>.js（便携式）                     │
                     └─────────────────────────────────────────────────────┘
```

仓库布局（新增 / 变更）：

| 路径 | 角色 |
|---|---|
| `packages/core/guide-format/` | **新增** — 快照类型、`parseGuideSnapshot`、`toPortableHtml`、`estimateBytes`、文件名 slug、fixtures。浏览器安全，零依赖（D5、D9）。 |
| `packages/guide-viewer/` | **新增** — 渲染器包 `@plannotator/guide-viewer`（D2）。像 `@plannotator/ui` 一样提供 TS 源码 + 预构建的 `styles.css`。 |
| `apps/guides-show/` | **新增** — 带静态资源的 Cloudflare Worker + `wrangler.toml` + 查看器 CDN 构建配置（D7、D8）。与 `apps/paste-service` 无关系。 |
| `packages/review-editor/` | 从 `@plannotator/guide-viewer` 导入引导链；保留 `GuideScreen`、`GuideEmptyState`、`GuideGenerating`（应用外壳）。 |
| `packages/server/`、`apps/pi-extension/server/` | 补丁捕获、`guide-store` v2、导出端点（D6）。 |
| `apps/hook/server/index.ts` | `guide export` 子命令（D9）。 |
| `.github/workflows/` | `guides-show-deploy.yml`，在发布标签上触发（D10）。 |

## 3. 快照格式（`@plannotator/core/guide-format`）— D5

```jsonc
{
  "kind": "plannotator-guided-review",
  "version": 1,
  "exportedAt": "2026-08-15T20:00:00Z",
  "guide": {
    "title": "Auth token refresh",
    "intent": "…markdown…",
    "sections": [{ "title": "…", "overview": "…markdown…", "diffs": [{ "file": "src/auth.ts", "summary": "…" }] }],
    "unplacedFiles": ["README.md"],
    "reviewed": [true, false]
  },
  "review": { "rawPatch": "diff --git a/… ", "gitRef": "main..HEAD", "diffType": "since-base", "base": "origin/main" },
  "source": {
    "kind": "pr",                              // local | pr | workspace | commit
    "repo": "backnotprop/plannotator", "branch": "feat/auth", "headSha": "…",
    "pr": { "url": "https://github.com/…/pull/1082", "number": 1082, "title": "…", "platform": "github" },
    "commitSha": null
  },
  "generator": { "engine": "claude", "model": "sonnet", "generatedAt": "…", "customInstructions": "…or null" },
  "theme": { "palette": "plannotator" }        // 可选提示
  // "annotations": 为 v2 预留（D3）— v1 解析器会拒绝它
}
```

规则：
- 严格：每一层的未知字段都以 JSON 路径错误拒绝（抢救的解析器）。`reviewed` 归一化为 `sections.length`。`guide` 为 `CodeGuideOutput & { reviewed }` — 绝不含 `saved`/`moved`。
- 无大小上限（D1）。仅有结构性健全性上限（sections ≤ 100，refs ≤ 50 000）以保证解析可终止。
- `languages` 是**派生**的，不存储：`detectLanguages(rawPatch)` 将文件扩展名映射到语法 id（复用 `getCallFlowPatchFiles` 风格的路径提取 + 一个小型 ext→lang 映射）。
- 每个版本一个 fixture，位于 `packages/core/guide-format/fixtures/v1/*.json`；`compat.test.ts` 用当前解析器解析所有 fixture（D8、D10 门禁）。

`toPortableHtml(snapshot, { viewer: { base, jsHash, cssHash, jsIntegrity, cssIntegrity, version } })` 输出：

```html
<!doctype html><html lang="en"><head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<meta name="plannotator-guided-review" content="v1">
<meta http-equiv="Content-Security-Policy" content="default-src 'none'; script-src https://guides.show 'wasm-unsafe-eval' blob:; style-src https://guides.show 'unsafe-inline'; font-src https://guides.show; img-src data: blob:; connect-src https://guides.show; worker-src blob:; base-uri 'none'; form-action 'none'">
<title>{title} · Guided Review</title>
<link rel="stylesheet" href="https://guides.show/v1/viewer.{cssHash}.css" integrity="sha384-…" crossorigin="anonymous">
<link rel="modulepreload" href="https://guides.show/v1/langs/typescript.{hash}.js" crossorigin="anonymous"> <!-- 每种检测到的语言一个 -->
</head><body>
<div id="root">
  <!-- D1/D6 回退：标题、意图、每节标题 + 概述 + 文件列表 + 摘要，渲染为简单 HTML。无网络也可读。 -->
</div>
<script id="plannotator-guided-review" type="application/json">{escaped snapshot}</script>
<script type="module" src="https://guides.show/v1/viewer.{jsHash}.js" integrity="sha384-…" crossorigin="anonymous"></script>
</body></html>
```

- 转义：JSON script 内的 `&`、`<`、`>`、U+2028/9 → `\uXXXX`（抢救的，已用补丁行中的字面 `</script>` 测试）。
- `connect-src https://guides.show` 仅用于动态导入语法块；不得获取其他任何内容（D4）。
- 文件名：`guided-review-{slug}.html`。

## 4. 查看器包（`@plannotator/guide-viewer`）— D2、D3、D4、D8

### 4.1 公开接口
```ts
export function GuideViewer(props: { snapshot: GuideSnapshot; theme?: 'light'|'dark'|'system'; palette?: string }): JSX.Element
export function mountGuideViewer(el: HTMLElement, snapshot: GuideSnapshot, opts?): { unmount(): void }
export { GuideView, GuideSectionCard, GuideFileCard, GuideViewportProvider } // 由 review-editor 使用
```
`GuideViewer` = 解析补丁（`parseDiffToFiles`）→ `ThemeProvider`（调色板来自 `theme` 提示 → cookie/系统）→ `GuideViewportProvider` → `GuideView`，带一个**只读**形态的 `codeViewProps` 包。任何地方都不 fetch；`configStore` 使用内存存储后端，因此在 `file://` 上不写 cookie。

### 4.2 重构（来自耦合审计；S/M 规模）
| # | 变更 | 位置 | 规模 |
|---|---|---|---|
| R1 | `GuideView` 将 `files`、可选 `revealFile`、可选 `activeSearchMatch` 作为 props（本地 reveal token 回退已存在） | `GuideView.tsx:65,79-83,107` | S |
| R2 | `GuideSectionCard` 接收 `allFiles` prop 而非 `state.files` | `GuideSectionCard.tsx:100` | S |
| R3 | `GuideFileCard` 接收一个 `codeViewProps` 包而非 46 个 `useReviewState` 读取 | `GuideFileCard.tsx:112-160` | M |
| R4 | 将 `renderInlineMarkdown.tsx`、`renderMarkdownProse.tsx`、`diffParser.ts`、`types.ts (DiffFile)` 移入包；将 `shared/diff-paths.ts` → `core` | 4 个文件 | S |
| R5 | 将 `shared/guide.ts`（类型）→ `core/guide`，保留 shim | — | S |
| R6 | 将 `usePierreTheme.ts:2` 类型导入重指向 `core/config-types` | — | S |
| R7 | `AllFilesCodeView` 的 `readOnly` prop：`enableLineSelection/enableGutterUtility=false`，无 `ToolbarHost`/`CommentPopover`/`FileCommentBanner`/`EditProvider`（置于 `React.lazy` 后以便代码分割出去） | `AllFilesCodeView.tsx:2290-2291, 2432-2489` | M |
| R8 | 将 `isActive`（标注目标）与 `enableKeyboardShortcuts`（`:2119` 处的 `window` keydown）拆分 | `AllFilesCodeView.tsx:2031-2033` | S |
| R9 | `FileHeader`：`OpenInAppButton` 置于显式 prop 后；`AllFilesCodeView`：`fileContentFetcher` 接缝（默认 = `/api/file-content`；查看器传 `null` → 无增强、无展开控件） | `FileHeader.tsx:332`、`AllFilesCodeView.tsx:1246` | M |
| R10 | 包 CSS 构建：Tailwind v4 `@source` 覆盖包 + `theme.css` 基础调色板 + 仅评审覆盖（`--text-xs: 0.8125rem`、`--panel-header-h`），镜像 `packages/ui/vite.css.config.ts` | 新增 | M |
| R11 | Shiki 收窄（见 4.3） | 构建 | M |
| R12 | review-editor 中的 `GuideViewAdapter`：读取 `useReviewState()`，构建包；`GuideScreen` 其余不变 | 新增，`GuideScreen.tsx:375-383` | S |
| R13 | 保留标注槽（D3）：`codeViewProps` 包保留 `annotations`/回调类型；只读宿主传空 — 不移除 API | — | S |

一致性门禁：现有 `GuideView.test.tsx` / `GuideSectionCard.test.tsx` 对包保持通过；应用渲染引导字节级一致（手动浏览器检查 + 在演示引导上对移动前后引导屏幕的 Playwright/DOM 快照）。

### 4.3 CDN 构建（`apps/guides-show/viewer.vite.config.ts`）— D8
- 入口 `viewer.tsx`：读取 `#plannotator-guided-review`、`parseGuideSnapshot`、`mountGuideViewer(#root)`；解析失败时在回退 body 上渲染错误卡片。
- **多文件构建，非单文件**：`inlineDynamicImports: false`。Shiki 的 `bundledLanguages` 条目已是惰性 `() => import(...)` 加载器，因此 Rollup 自动为**每种语法输出一个 chunk** → `/v1/langs/<lang>.<hash>.js`。复用 `apps/review/vite.config.ts` 中的 `shiki/wasm` 别名。**Spike（阶段 2 门禁）：** 确认 `@pierre/diffs` 的静态 `bundledLanguages` 导入保持映射很小且语法在 chunk 中；若否，将 `shiki` 别名到一个导出相同映射形状的收窄 shim。
- 字体：`@fontsource-variable/inter` + `geist-mono` 拉丁子集作为文件输出到 `/v1/fonts/`，由 CSS 引用；`font-display: swap` 带系统回退（D8）。
- Worker：将 `worker.<hash>.js` 输出为真实的、无导入的文件（`inlineDynamicImports`；`check-budgets` 断言无模块语法，以便作为经典 worker 运行）。运行时（`viewer/portablePool.tsx`）：fetch 它，然后按序探测构造策略 — blob classic、blob module、data classic、data module — 用一个必须应答的单行 worker；第一个应答的构建池，都无 → 主线程。Chrome 拒绝来自 `file://` 的 blob module worker（"cross-origin redirects of the top-level worker script"，通过 `error` 事件异步），但接受经典 blob worker，这就是为何探测是实时的而非 `location.protocol` 切换。若池在 4 秒内未初始化，`PoolWatchdog` 丢弃 provider（Pierre 在等待 worker 已死的池时不渲染任何内容），引导在主线程重新渲染。
- 输出清单 `manifest.json` `{ version:1, js, css, integrity:{js,css}, langs:{ts:'langs/typescript.<h>.js',…} }`，由 `toPortableHtml` 的调用方消费（服务器/CLI 在构建时读取；Plannotator 二进制嵌入其发布时所用查看器的清单）。
- **大小预算（CI 超出即失败）：** `viewer.js` ≤ 1.2 MB 原始 / 400 KB gzip；`viewer.css` ≤ 120 KB / 30 KB gzip；HTML 外壳开销（除快照外的一切）≤ 25 KB。在阶段 2 后测量并收紧。

## 5. guides.show — D7、D8、D10

- `apps/guides-show/`：Cloudflare Worker（静态资源绑定）+ `wrangler.toml`（自有账户/项目；**与 `apps/paste-service` 无共享代码或配置**）。
- 路由：`/v1/*` 静态，`Cache-Control: public, max-age=31536000, immutable`，`Access-Control-Allow-Origin: *`（来自 `file://` 和其他源的语法/字体/worker 获取），`Cross-Origin-Resource-Policy: cross-origin`。`/g/*` → 404 JSON `{ reserved: true }`（路由预留，D11）。`/` → 最小落地页。
- 不可变性：部署步骤**只添加**文件；绝不覆盖现有 `/v1/<name>`（上传脚本检查存在性，内容不同则哈希冲突失败）。
- 部署：`.github/workflows/guides-show-deploy.yml` 在 `v*` 标签上 — 构建查看器 → 运行 `compat.test.ts` → 上传资源 → 将清单发布为 release 资源。本地：`bun run --cwd apps/guides-show deploy`。
- 平台就绪（D7）：Worker 骨架保留路由器，以便 `/g/<id>` + 上传可作为路由添加；R2 绑定已声明但未使用；现在无 KV/DO。

## 6. Plannotator 生产者 — D6、D9、D11

### 6.1 启动时补丁捕获（Bun + Pi）
- `buildCommand` 引导分支（`review.ts:~1060-1198`）在 `changedFilesSnapshot`/`guideContext` 旁返回 `launchReview: { rawPatch, gitRef, diffType, base, source }`。**修复复用失败任务的 `launchReview`。**
- `agent-jobs.ts`