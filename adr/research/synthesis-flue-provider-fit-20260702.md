# 综合分析：Flue 作为 Plannotator 的 agent-job 提供方

日期：2026-07-02
主题：Flue 的本地检出（`@flue/runtime` 1.0.0-beta.1，Apache-2.0）
方法：浅层广度调研 + 三份深度子代理报告（运行时执行模型；CLI 无头调用；SDK/HTTP 接口 + 运维）。

## Flue 是什么（以及不是什么）

Flue 是一个 TypeScript **agent-harness 框架**：你用代码编写 agent/工作流（`createAgent`、valibot 类型化的工具/技能、沙箱），`flue` CLI 将它们构建（Vite 代码生成）为 Hono 服务器，其自身的循环（通过 `@earendil-works/pi-agent-core` / `pi-ai`）直接调用模型 API。它不是 claude/codex/cursor/opencode 那样的编码 agent CLI——没有第三方 agent 可供委派。采用 Flue 意味着 Plannotator 要自行编写并维护自己的小型 agent。

## 结论

**方案 A（派生式 job 提供方）：可行，但有实际成本。**
**方案 B（Ask-AI 会话提供方）：目前不可行。**

## 关键事实

### 支持（方案 A）
- **真正的一次性执行存在**：`flue run <workflow> --target node --payload '{...}'` 会构建、运行工作流至完成，向 stdout 打印一个干净的 JSON blob（工作流的返回值），进度输出到 stderr，退出码 0/1/130/143。代码（bin/flue.ts:1361-1404）和 GitHub Actions 部署文档均可佐证。
- **一流的结构化输出**：`session.prompt(text, { result: <valibot schema> })` 会注入由 schema 派生的 `finish`/`give_up` 工具，用原始 schema 校验，并**将校验失败反馈给模型重试**（result.ts:159-303）。强于 claude 的 `--json-schema`，远强于我们的 nonce 标记契约——无需标记回退。
- **任意仓库 cwd 已确认**：`local({ cwd })` 将 bash/read/write/edit/grep/glob 绑定到任意绝对路径，具有真实的 child_process 语义；中止/超时时按进程组终止。环境变量为白名单（密钥必须显式转发）。
- **实时日志**：我们的 jobs 引擎已将 stderr 流式写入 `job:log`——Flue 的 ANSI 进度文本今天就能流过（需剥离 ANSI；它不是 NDJSON，因此若没有针对派生服务器的双进程 `flue logs --format ndjson` 操作，就没有结构化日志事件）。
- Apache-2.0，无遥测/回连。

### 反对 / 成本
1. **我们成为 agent 作者。** 一个内嵌的 Flue 项目（一个工作流、一个 agent 定义、提示词管道、只读工具策略）成为 Plannotator 需维护的代码——而今天的提供方中，agent 行为随第三方 CLI 一起发布。
2. **认证倒退**：pi-ai 从环境变量读取原始 API 密钥（`ANTHROPIC_API_KEY` 等）或 `ANTHROPIC_OAUTH_TOKEN`（优先）。**没有任何代码路径借用用户已登录的 Claude Code / Codex CLI 凭据。** 今天的提供方依赖现有 CLI 登录；Flue 提供方将要求用户管理密钥（或我方进行脆弱的令牌提取）。
3. **运行时 + 分发负担**：要求 Node ≥22.18，**明确拒绝 Bun**（对派生无碍，但成为用户机器上的新依赖）；内嵌项目需要安装 node_modules 并进行 Vite 构建（每次 `flue run` 都会重建；可通过在安装时预构建并派生 `node dist/server.mjs` + 一次 `POST /workflows/:name?wait=result` 来避免）。已有先例：`plannotator install-runtime agent-terminal` 已在管理 Node 运行时安装。
4. **只读强制是定制工作**：没有标志可禁用 write/edit/bash；需要自定义 `SandboxFactory.tools`，通过 `defineTool()` 重新实现 read/grep/glob（Flue 自身的工具实现未导出）。
5. **Beta 频繁变动**：1.0.0-beta.1（2026-06-16）之前是快速破坏性的 0.x 节奏；进程内嵌入接缝（`createFlueContext`）明确为内部接口。
6. **模型目录**：没有 `flue models`；我们需要针对 pi-ai 的目录整理一份列表。

### 方案 B 为何不可行（目前）
- `@flue/sdk` 是面向 Flue 自有服务器的 HTTP 客户端，不是可嵌入的运行时；会话构造 API 位于 `@flue/runtime/internal`，标注"切勿从此处导入"。
- **没有工具权限/人在回路 Hook**——工具事件仅用于观测；我们的交互式审批提示没有对应物。
- **没有通过 HTTP 的服务端中止**——断开连接不会停止运行（设计上为持久化准入）。
- `examples/pi-provider-chat` 与我们的 Pi 集成无关（它演示的是 pi-ai 的 OAuth 支持的模型提供方）。

## 若采用：最小集成草图

- 内嵌项目 `packages/server/flue-provider/`（flue.config.ts，一个 `workflows/run-job.ts` 接收 `{ prompt, repoPath, schemaKind }`，agent 使用 `local({ cwd: repoPath })` + 精选只读工具集），在安装时构建。
- 在 `SERVER_BUILT_PROVIDERS` 中新增提供方 `flue` + 能力（node ≥22.18 + 已构建项目存在 + 已配置的 API 密钥/OAuth 令牌）。
- `buildCommand`：每个 job 派生预构建的 `node dist/server.mjs`（临时端口）+ `POST /workflows/run-job?wait=result`；或为 v1 简洁性接受 `flue run` 的重建成本（stdout JSON 解析，stderr → 日志）。
- 复用：guide/tour/review schema 作为工作流的 valibot `result` schema 透传——单一输出管道，无标记解析。

## 建议

**今天**不值得作为 claude/codex/cursor/opencode 的同级采用：认证模型（原始 API 密钥 vs. 依赖用户 CLI 登录）和自写 agent 的负担超过了结构化输出的优势，且 API 在 1.0 之前仍在变动。若/当以下情况出现时重新评估：(a) Flue 发布公开的一次性/嵌入 API，或稳定的 `flue run` 并提供机器可读事件输出；(b) pi-ai 支持从已安装 CLI 借用凭据；或 (c) 我们极度需要一个完全自有、schema 原生的 job agent，以至于愿意出资维护——届时 Flue 将是我们评估过的最强 harness 候选。