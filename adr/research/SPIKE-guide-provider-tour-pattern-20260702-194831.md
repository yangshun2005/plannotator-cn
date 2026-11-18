# SPIKE：以 Code Tour provider 作为 Guide provider 的模板

日期：2026-07-02
状态：已对照代码验证。为 `adr/decisions/006-guided-review-first-class-feature-20260702-192821.md` 的规格工作提供输入。

## 问题

ADR 006 指出，guide agent-job provider 应遵循 Tour 模式（`packages/server/tour/`），"而不是发明新的生命周期"。本 spike 端到端记录该模式，附精确的 file:line 引用，使 guide provider 的规格可以直接引用一份检查清单，而无需重新推导。

## 1. `packages/server/tour/tour-review.ts` —— provider 模块

单个文件负责：系统提示词、JSON schema、用户消息构建器、两个引擎的 CLI 命令构建器、输出解析器，以及一个 session 对象——它把上述全部内容封装在一个小接口后面，供 jobs 引擎调用。

- **`TOUR_SCHEMA_JSON`**（:20-84）—— 手工构建的 JSON Schema 字符串（并非从 TS 类型派生），处处 `additionalProperties: false`，原样传给两个 CLI 的 schema 参数。结构：`title, greeting, intent, before, after, key_takeaways[], stops[], qa_checklist[]`。每个 `stop` 含 `title/gist/detail/transition/anchors[]`；每个 `anchor` 含 `file/line/end_line/hunk/label` —— **`hunk` 是模型必须根据所读 patch 自行编造的完整 unified-diff 字符串**，而非指向真实 patch 的指针。这是值得标记的刻意 schema 选择（见 §7）。

- **`TOUR_REVIEW_PROMPT`**（:86-287）—— 一段长而精心调校的系统提示词：身份/语气规则（对话式、不用破折号、不用 emoji）、逐字段讲解每个 schema 字段并附示例的 "Output structure" 部分、"Pipeline" 部分（读 diff → 读 CLAUDE.md/README → 读提交/PR → 归组为 stops → 按阅读流排序 → 输出 JSON），以及重申不可协商项的 "Hard constraints" 部分（hunk 非空、不编造行号、gist 一句话等）。ADR 006 明确说 guide 提示词"获得同等程度的打磨"——这就是标准。

- **`buildTourUserMessage`**（:289-338）—— 按上下文形态分派：PR 全栈、PR 有本地 worktree 访问、PR 无本地访问、workspace（多仓库）、或本地 diff（通过 `../agent-review-message` 的 `getLocalDiffInstruction`）。每个分支产出一段简短指令 +（有时）围栏 diff 块中内联的 `patch`。`buildWorkspaceTourUserMessage`（:340-353）是多仓库变体。

- **`buildTourClaudeCommand`**（:360-404）—— Claude 引擎的精确 argv：
  ```
  claude -p --permission-mode dontAsk --output-format stream-json --verbose
    --json-schema <TOUR_SCHEMA_JSON> --no-session-persistence
    --model <model> [--effort <effort>]
    --tools Agent,Bash,Read,Glob,Grep
    --allowedTools <逗号列表：Agent,Read,Glob,Grep,git/jj/gh/glab 只读子命令,wc>
    --disallowedTools <Edit,Write,NotebookEdit,WebFetch,WebSearch,解释器,curl/wget>
  ```
  提示词经 stdin（`stdinPrompt`）传入，而非 argv——避免 argv 长度/转义限制。上游必须设 `captureStdout: true`，以便 jobs 引擎缓冲 stdout 供 `parseTourStreamOutput` 使用。

- **`buildTourCodexCommand`**（:423-449）—— Codex 路径把 JSON schema 写入物化文件（`ensureTourSchemaFile`，:410-417，做了记忆化，每进程只写一次），因为 Codex 接受 `--output-schema <path>`，而非内联 schema。全局标志（`-m`、`-c model_reasoning_effort=`、`-c service_tier=fast`）必须位于 `exec` 子命令之前。输出经 `-o <outputPath>`（`generateTourOutputPath`，:419-421，使用 `os.tmpdir()` + uuid）写入临时文件，而非 stdout——`--full-auto --ephemeral -C <cwd> <prompt>`。

- **解析器**：
  - `parseTourStreamOutput`（:451-475）—— **倒序**遍历 stdout 各行，逐行按 JSON 解析，查找 `{ type: "result" }`，遇 `is_error` 即放弃，并且——关键地——**若 `stops` 缺失/为空则拒绝解析**（注释："A tour with no stops isn't a tour"）。任何失败返回 `null`；从不抛异常。
  - `parseTourFileOutput`（:477-491）—— Codex 文件输出路径同样有空的 stops 守卫；成功和 catch 路径都始终尝试 `unlink(outputPath)`（临时文件清理在此处完成，不在 jobs 引擎）。
  - `TOUR_EMPTY_OUTPUT_ERROR`（:18）—— 当解析在结构上成功（exit 0）但产出 `null` 时，向 job 呈现的平缓字符串。

- **`createTourSession()`**（:542-604）—— review.ts 持有其单个实例的对象（review.ts:206 的 `const tour = createTourSession()`）。两个内存 `Map`（`tourResults: Map<jobId, CodeTourOutput>`、`tourChecklists: Map<jobId, boolean[]>`）—— **无持久化，随服务器进程消亡**。暴露：
  - `buildCommand({ cwd, patch, diffType, options, prMetadata, config })` —— 读取 `config.engine`（默认 `"claude"`）、`config.model`（Claude 回退到 `"sonnet"`，Codex 留空——注释解释原因：`"sonnet"` 是 Claude 专属模型 id，不能泄漏给 Codex）、`config.effort`、`config.reasoningEffort`、`config.fastMode`。构建一次用户消息，前置 `TOUR_REVIEW_PROMPT`，然后按引擎分支 → 返回 `TourSessionBuildCommandResult` 结构（command/outputPath/stdinPrompt/cwd/label/prompt/engine/model/effort/reasoningEffort/fastMode）。
  - `onJobComplete({ job, meta })` —— 按 `job.engine === "codex" ? outputPath : stdout` 选择解析器，把结果存入 `tourResults`，返回 `TourSessionJobSummary`（`correctness: "Tour Generated"`、`explanation: "<n> stops, <m> QA items"`、`confidence: 1.0`）。解析失败返回 `{ summary: null }`——不抛异常，不部分写入。
  - `getTour(jobId)` —— 合并存储的 `CodeTourOutput` 与其 checklist（默认 `[]`），若该 job 结果从未存储则返回 `null`。
  - `saveChecklist(jobId, checked)` —— 盲覆盖，不校验长度是否匹配 `qa_checklist.length`。

## 2. `packages/server/review.ts` 中的 jobs 引擎集成

- `const tour = createTourSession();` —— review.ts:206。每个 review 服务器进程/session 一个实例（符合 ADR 006 明确推迟的"guide 持久化超出服务器内存"）。

- `createAgentJobHandler({ mode: "review", ..., buildCommand, onJobComplete })` —— review.ts:616 打开整个 handler；外层 `buildCommand` 闭包（:621-752）**在任何 `await` 之前做状态快照**（:622-624 注释："Snapshot ALL launch-relevant state before any await: waiting out the checkout warmup below yields to other requests"）。它预先把 `prMetadata`、`currentPatch`、`currentDiffType`、`currentBase`、`currentPRDiffScope` 快照进 `launch*` 局部变量，解析请求的 review profile，然后（仅 PR 模式相关）在设置了 `options.worktreePool` 时 await `ensurePRLocalCwd(launchMetadata)`——若 PR 检出尚未就绪则抛出面向用户的错误（"Local PR checkout unavailable... Retry shortly"），而不是静默在错误目录中运行。随后构建 `diffContext`（mode/base/worktreePath，或 workspace 模式下 worktreePath 为 `null`，或 PR 模式下完全 `undefined`，因为 `prMetadata` 已覆盖）——该快照随每个 provider 的 job 传递，tour 也不例外。

  **`if (provider === "tour")` 分支**（:690-700）是外层 switch 中的第一个检查（在 codex/claude/marker 分支之前），只是转发到 `tour.buildCommand({ cwd, patch: launchPatch, diffType: launchDiffType, options: userMessageOptions, prMetadata: launchMetadata, config })`，然后把 `prUrl/diffScope/diffContext/reviewProfileId/reviewProfileLabel` 拼接到 `tour.buildCommand` 返回的内容上。Tour 是*唯一*把整个 `buildCommand` 主体委托给外部模块的 provider——codex/claude/marker 分支在 review.ts 中内联各自逻辑。

- **`onJobComplete`**（:754-915）是一个大函数，末尾按 provider 分支。**`if (job.provider === "tour")` 分支**（:901-914）调用 `tour.onJobComplete({ job, meta })`。若返回 `summary`，则赋给 `job.summary`（与其他 provider 相同的字段——codex/claude/marker 内联设置 `correctness/explanation/confidence`，tour 的只是 `{correctness: "Tour Generated", explanation: "<n> stops...", confidence: 1.0}`）。若 `summary` 为 `null`，该分支**翻转 `job.status = "failed"`** 并设置 `job.error = TOUR_EMPTY_OUTPUT_ERROR`——明确注释："the client doesn't auto-open a successful-looking card that 404s on `/api/tour/:id`." 这是其上方 marker-engine 分支在自己注释（:854）中明确引用的 fail-closed 先例。

- **路由** —— review.ts:1015-1034，在 `fetch()` handler 中内联注册，位于通用 `/api/diff` 路由*之前*：
  - `GET /api/tour/:jobId`（:1016-1021）—— 正则匹配，`tour.getTour(jobId)`，未命中返回 404 `{ error: "Tour not found" }`，否则返回完整 `CodeTourOutput & { checklist }` 的 JSON。
  - `PUT /api/tour/:jobId/checklist`（:1024-1034）—— 正则匹配，解析 `{ checked: boolean[] }`，仅当 `checked` 是数组时调用 `tour.saveChecklist`，返回 `{ ok: true }` 或 JSON 错误时 400。不校验 `checked.length` 是否匹配 tour 的 `qa_checklist.length`。

## 3. `packages/server/agent-jobs.ts` —— 通用 jobs 引擎触点

- **`SERVER_BUILT_PROVIDERS`**（:57-63）—— `Set(["claude", "codex", "tour", "cursor", "opencode"])`。在此集合中意味着：客户端提供的 `command` argv 被丢弃（:568-571），`null`/缺失的 `buildCommand` 结果是硬错误（:561-567），绝不静默回退到客户端 argv。**guide provider 必须加入此集合**，否则 `/api/agents/jobs POST` handler 会接受并启动任意客户端提供的命令。

- **能力条目**（:158-162）—— `{ id: "tour", name: "Code Tour", available: !!Bun.which("claude") || !!Bun.which("codex") }`。可用性跨该 provider 可用的两个引擎做 OR（不同于 codex/claude 能力各自只依赖自己的二进制）。若 guide 条目也支持两个引擎，则遵循同样的 OR 形态。

- **`AgentJobHandlerOptions.buildCommand` 返回结构**（:81-110）—— provider 构建器可填充的完整可选字段集：`command/outputPath/captureStdout/stdinPrompt/cwd/label/prompt/engine/model/effort/reasoningEffort/fastMode/prUrl/diffScope/diffContext/reviewProfileId/reviewProfileLabel`。Tour 填充了除 `reviewProfileId/Label` 之外的所有字段——这两个来自外层 review.ts 包装器，而非 `tour.buildCommand` 本身（tour.ts 不知道 review profile）。

- **`spawnJob`**（:202-412）—— 通用进程生命周期。对新 provider 值得注意：
  - stdout 捕获通过 `captureStdout` 选择加入（:237、:244）—— 写文件的 Codex 风格 provider 不需要；Claude 引擎的 tour job *确实*设置它（镜像 Claude review 路径）。
  - stdout 格式化的 `emitLogLine` 闭包（:310-333）有明确的 tour 特例："Tour jobs with the Claude engine also stream Claude JSONL, so key off engine too"（:312-314）—— 检查 `provider === "claude" || spawnOptions?.engine === "claude"` 以走 `formatClaudeLogEvent`。**使用 Claude 引擎的 guide provider 需要添加相同的 `|| spawnOptions?.engine === "claude"` 检查**，否则其实时日志流会渲染原始 JSONL 而非格式化文本。
  - 退出处理（:354-398）仅在 `exitCode === 0` 时调用 `onJobComplete`；tour 的 fail-closed（空输出时 `job.status = "failed"`）发生在 `onJobComplete` *内部*，不在此处——通用引擎仍先标记 job 为 `"done"`（:364），并让 provider 的 `onJobComplete` 在摄取同步运行后、`job:completed` 广播（:395）之前将其降级。

- **`POST /api/agents/jobs` 请求体校验**（:520-653）—— `KNOWN_JOB_FIELDS`（:527-531）为 `["provider", "command", "label", "engine", "model", "reasoningEffort", "effort", "fastMode", "reviewProfileId"]`；请求体中任何其他键都会 400。**若 guide 启动需要此列表中没有的 provider 专属字段（鉴于 tour 不需要任何字段，这不太可能），也必须在此添加。**

## 4. 共享类型

- **`packages/shared/tour.ts`**（61 行，已通读全文）—— 纯接口，无逻辑：`TourDiffAnchor`、`TourKeyTakeaway`、`TourStop`、`TourQAItem`、`CodeTourOutput`（agent 的原始输出），以及 `CodeTourData = CodeTourOutput & { checklist: boolean[] }`（合并 checklist 后的 UI 侧形态）。guide 等价物（`GuidePage`、`GuideDiffSlice`?、`GuideOutput`、`GuideData = GuideOutput & { reviewed: boolean[] }`）是直接对应。

- **`packages/shared/agent-jobs.ts`**（156 行，已通读全文）—— 与 provider 无关的 job 模型。`AgentJobInfo` 上的相关字段（:31-82）：`provider: string`（自由格式 id，当前为 "tour"）、`engine?: string`（:39 文档注释称"当 provider 为 'tour' 时设置"——**此注释需更新以提及 guide**）、`model/effort/reasoningEffort/fastMode`（均与 provider 无关，tour 原样复用）。`source` 为 `jobSource(id) = "agent-" + id.slice(0,8)`（:141-143）—— 用于标记 job 创建的注释；guide provider 可能不直接摄取注释（guide 页面不是发现项），因此 `source` 对其可能未使用，这没问题——它仍由 `spawnJob` 统一赋值。`markJobReviewFailed`（:153-156）是 Tour 先例的 fail-closed 辅助函数，明确注释为可复用（"Code Tour precedent"）—— review.ts 中的 marker-engine 路径手工内联了相同模式而非调用此辅助函数（添加 guide 时值得规范化，非必需）。

## 5. 客户端

- **`packages/review-editor/hooks/tour/useTourData.ts`**（全文，107 行）—— `useTourData(jobId)` hook：挂载时请求 `/api/tour/:jobId`（当 `jobId === DEMO_TOUR_ID` 时短路到 `DEMO_TOUR`，:29-35），从 `data.checklist` 或按 `qa_checklist.length` 大小的全 false 数组派生初始 `checked`（:44），并防抖持久化 checklist 切换（500ms，:57-72），外加卸载时同步 `keepalive: true` 刷新（:89-103），以免关闭标签页丢失最后一次切换。guide 的"Reviewed"复选框 hook 是此结构克隆，大小按页面数而非 QA 项数，请求 `PUT /api/guide/:jobId/reviewed` 形态的端点。

- **`demoTour.ts`** —— 开发模式 fixture（`DEMO_TOUR_ID = 'demo-tour'`、`DEMO_TOUR: CodeTourData`），无需实时 agent job 即可渲染，开发模式下由 Cmd/Ctrl+Shift+T 切换（App.tsx :766-777）。在 agent/schema 存在之前迭代 guide 接管屏视觉设计时是有用先例。

- **`App.tsx`** 对话框生命周期接线（除所有 `import.meta.env.DEV` 门控部分外）：
  - `tourDialogJobId` 状态（:352-353）+ `handleOpenTour`