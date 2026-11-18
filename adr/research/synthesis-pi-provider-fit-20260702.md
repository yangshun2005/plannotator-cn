# 综合评估：Pi 作为 Plannotator 的 agent-job 提供方

日期：2026-07-02
主题：Pi 的本地检出（`@earendil-works/pi-coding-agent` 0.80.3，MIT）
方法：浅层广度调研 + 三份深度子代理报告（CLI 无头模式；可扩展性/SDK/认证；Plannotator 侧集成接缝）。
配套文档：`synthesis-flue-provider-fit-20260702.md`（Flue 结论：通过）。

## 结论

**可以。Pi 适合作为 agent-job 提供方，且比当前的 cursor/opencode 更干净。** 提出此需求的用户已通过 Pi 认证，因此零配置。推荐形态：v1 作为第三个 marker 引擎，并保留通过 Pi 官方 finish-tool 扩展升级到 schema 原生输出的直通路径。

## 为何契合（逐条对照契约）

| Plannotator job 契约 | Pi 的答案 |
|---|---|
| 一次性无头启动 | `pi -p "<prompt>"` / `--mode json`——完整工具循环，完成后退出（print-mode.ts） |
| 从 stdout NDJSON 获取实时日志 | `--mode json`：每行一个 JSON 事件（会话头、`agent_start` → `message_update` 增量 → `tool_execution_*` → `agent_end`）——与 Claude 的 stream-json 同属一类形态 |
| 结构化 JSON 结果 | v1：沿用现有 nonce marker-block 契约，从最终 assistant 消息文本中提取。v2：**schema 原生**，通过随附扩展（`pi -e finish-tool.ts`）注册一个运行时校验、带 `terminate: true` 的工具——Pi 以 `examples/extensions/structured-output.ts` 原样提供此模式；参数非法会产生逐字段错误，模型必须重试。强于 marker，可比 claude `--json-schema` |
| "开箱即用"的认证 | 透明复用 `~/.pi/agent/auth.json`：Claude Pro/Max OAuth、ChatGPT Plus/Codex、GitHub Copilot 或 API key，自动刷新。无需新增密钥管理 |
| cwd = 仓库 | 作用于启动时的 cwd（无 `--cwd` 标志；我们本就以 cwd 启动） |
| tour/guide 只读 | `--tools read,grep,find,ls`——在工具注册层强制，直接禁用 edit/write/bash（强于我们为 cursor/opencode 提供的任何方案） |
| 模型选择 | `--model provider/id[:thinking]`、`--provider`、`--thinking off..xhigh` |
| 模型目录 | `--list-models` 是人类可读表格（无 JSON）。方案：在 `parseModels` 中解析表格，或复用 `packages/ai/providers/pi-sdk.ts:247-281` 中现有的 `pi --mode rpc` + `get_available_models` 机制（已构建、已按认证过滤） |
| 会话卫生 | `--no-session`（临时）或 `--session-id <job-id>`（未来"查看 agent 轨迹"功能） |
| 终止/中止 | SIGTERM → 退出码 143；进程级，与其他提供方一致 |
| 能力检测 | `Bun.which("pi")`——我们已为 Ask-AI 提供方这样做 |

## 集成中需编码处理的坑（均已确认）

1. **`--mode json` 即使运行失败也退出 0。** 文本模式的 stopReason 检查以 `mode === "text"` 为条件（print-mode.ts:129）。失败检测必须来自事件流：最后一条 assistant 消息的 `stopReason: "error"/"aborted"`（+ `errorMessage`），或缺失 `agent_end`。退出 1 仅表示 Pi 自身崩溃/配置错误（无认证、模型错误、扩展加载失败）。
2. **项目信任**：非交互运行会静默应用 `defaultProjectTrust`。对任意检出的 job 始终传 `--no-approve`，确保 `.pi/settings.json`、项目扩展和项目 skills 永不加载——不要依赖默认值，用户的全局设置或此前的 `/trust` 可能翻转它。
   （v2 验证：宿主随附的 `-e` 扩展在 `--no-approve` 下必须仍能加载；它是 CLI 资源而非项目资源，但发布前需确认。）
3. **无 job 超时标志**——我们自行强制墙钟时间和 SIGTERM（现有姿态）。
4. **Stdin 合并怪癖**：管道 stdin 会无分隔符地拼接到 argv prompt 后（initial-message.ts:20-43）——若我们管道传入 diff 上下文，需自行追加换行。
5. **发布节奏近乎每日（0.80.x）**——将未文档化的内部实现视为不稳定；我们的接触面（CLI 标志 + JSON 事件）有文档，但预期只锚定文档化行为。
6. **Pi 中根本不存在权限提示**（设计如此；README："No permission popups. Run in a container"）。对 job（即发即忘）无关紧要，但若我们加深 Ask-AI 提供方则相关。

## 集成草图（v1：第三个 marker 引擎）

以下所有接缝均已在 Plannotator 侧报告中以 file:line 引用映射：

1. `packages/server/marker-review.ts`——添加 `pi` 的 `MarkerEngine` 描述符（六个字段）：`binary: "pi"`、`buildArgv: ["--mode","json","--no-session","--no-approve",...model flags,"-p",prompt]`、`extractText` 从 Pi 的 `message_end`/`message_update` 事件读取 assistant 文本、`formatLogEvent` 处理相同事件、`modelsArgv`/`parseModels`（表格解析，或将能力接到现有 pi-sdk `fetchModels`）。对 tour/guide 只读运行，argv 增加 `--tools read,grep,find,ls`——注意 `buildArgv` 目前不接受只读参数；需穿线传入或在 guide/tour 分支处理。
2. `packages/server/agent-jobs.ts:57-64`——将 `"pi"` 加入 `SERVER_BUILT_PROVIDERS`。能力条目 + 惰性模型发现通过 `MARKER_ENGINES` 循环免费获得。
3. `packages/server/review.ts`——放宽两处 `as "cursor"|"opencode"` 类型断言；marker 兜底分支零新增代码即可处理 `pi`。
4. `packages/server/guide/guide-review.ts`——放宽引擎联合类型 + 两处类型断言（guide 已支持 marker 引擎）。
5. `packages/server/tour/tour-review.ts`——tour 目前仅支持 claude/codex；将 pi 加入 tour 意味着先给 tour 添加 marker 引擎支持（与 cursor/opencode 相同的缺口——单独决策）。
6. UI：`useAgentSettings` 的 `PiSection`（扁平 `{model}`，cursor 风格），放宽 `ReviewEngine`/`REVIEW_ENGINES`；`AgentsTab` 标签/图标映射（Ask-AI 提供方已有 `PiIcon`）、`piAvailable`、cursor 风格模型目录接线；`GuideEmptyState` 的 `ENGINE_LABEL` 条目（引擎列表由此派生）。
7. Pi 镜像：`marker-review.ts` 自动 vendor；手动镜像 `SERVER_BUILT_PROVIDERS` + 类型断言放宽到 `apps/pi-extension/server/{agent-jobs,serverReview}.ts`。注意递归：Pi 扩展将能在 Pi 会话内启动嵌套的 `pi` job——架构上等同于今天在 Claude Code 内启动 cursor；子进程像每个 job 一样继承 `PLANNOTATOR_AGENT_SOURCE`/`PLANNOTATOR_API_URL`。无需特殊处理，值得一个测试。

**v2 升级（schema 原生）**：按 job schema（review findings / guide / tour）随附一个小型 finish-tool 扩展，用 `-e` 加载，从 `tool_execution_end` 解析终止工具的校验结果，而非 marker 文本。这将 Pi 移出 marker 兜底进入自己的分支（如 claude/codex）——待 v1 证明需求后再做。

## Pi vs Flue（为何此选为"是"）

| | Pi | Flue |
|---|---|---|
| 是可委托的编码 agent | 是（其全部意义） | 否（harness 框架；需我们编写 agent） |
| 无头一次性 | `-p` / `--mode json` | `flue run`（需内嵌项目 + Vite 构建） |
| 结构化输出 | 现在 marker，后续官方 finish-tool 扩展 | valibot 原生（最佳）但需项目编写 |
| 认证 | 沿用用户现有 Pi 登录（OAuth 订阅或 key） | 原始 API key / 手动 token 环境变量 |
| 运行时 | 纯 Node CLI，可从 Bun 启动；除 `pi` 本身外无安装负担 | Node ≥22.18 + node_modules + 构建步骤；Bun 被拒 |
| 许可证 / 成熟度 | MIT，0.80.3，每日发布 | Apache-2.0，1.0.0-beta.1，破坏性变动频繁 |
| 契合现有接缝 | 几乎原样落入 MARKER_ENGINES | 需全新定制提供方机制 |

## 建议

放行 v1（marker 引擎）。这是在成熟接缝上的受控改动，服务真实用户需求，对 Pi 用户零配置，且只读强制优于现有 marker 引擎。tour 支持（需先有 tour marker 管道）和 schema 原生扩展推迟到快速跟进。