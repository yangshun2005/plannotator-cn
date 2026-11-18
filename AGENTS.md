# Plannotator 智能体指南

Plannotator 是 Claude Code 的计划审查 UI，通过 Hook 拦截 `ExitPlanMode`，让用户批准或请求修改并附上批注反馈。它还提供 git diff 的代码审查，以及任意 Markdown 文件的批注功能。

本文件是地图。完整指南位于 [`docs/agent-guides/`](docs/agent-guides/) 下的专题指南中。先读下面的硬性规则，再读你正在改动的代码或行为对应的指南。`CLAUDE.md` 是指向本文件的符号链接；`packages/ui/AGENTS.md` 补充了仅适用于 `packages/ui/` 的指南。

## 硬性规则（动手前必读）

- **绝不牺牲 UX。** Plannotator 是开源产品，必须好用、有魔法般的体验；它不是企业关键系统。不要出于防御目的增加摩擦：不加额外的门禁、确认、仅所有者可操作的检查、警告或设置来让产品更难用。修复真实 bug 时不要让产品变得更难用；如果某个安全想法会损害 UX，先问所有者。
- **一个人的多台设备体验一致。** 每个 Inbox 功能在他人已关联的另一台机器上的会话、智能体或项目上，与本机行为完全相同：打标签、Priority、置顶、启动智能体、回复，任何操作都一样。绝不发布只在本机会话上生效的功能，也绝不添加区别对待已关联机器会话的规则。把操作发送给拥有该对象的机器；如果旧版本或离线机器暂时无法接收，这是唯一允许的缺口，该机器更新后即关闭。完成功能前，两种方式（两个已关联 Inbox）都要构建并测试。见 [已关联机器](docs/agent-guides/inbox-machines.md)。
- **在商业版 Workspaces 应用中复用文档 UI？** 先读 [`packages/ui/README.md`](packages/ui/README.md)：此前从零重建该 UI 曾导致应用故障并被回滚。不要重建 UI，也不要重建 `packages/document-ui`；改为给 `@plannotator/ui` 加接缝（`configurePlannotatorUI()`），保持 Plannotator 应用不变，在人工于浏览器中确认功能对等前，绝不删除可用代码。
- **两个服务端运行时同步修改。** 端点变更要同时写入 Bun 服务端（`packages/server/`）和 Pi 服务端（`apps/pi-extension/server/`）；共享逻辑放在 `packages/shared/`。只有 Inbox 和 Shapes 服务端是 Bun 专属。见 [服务端运行时](docs/agent-guides/project-and-development.md#server-runtimes)。
- **构建顺序：review，然后 hook，然后 opencode。** `bun run --cwd apps/review build && bun run build:hook && bun run build:opencode`；顺序错误会复制过期的 HTML。见 [构建](docs/agent-guides/project-and-development.md#build)。
- **Tailwind `@source`。** 新目录中若有使用 Tailwind 类的 `.tsx` 文件，需要在应用的 `index.css` 中添加对应的 `@source` 条目。
- **测试规则**（完整列表见 [测试规则](docs/agent-guides/project-and-development.md#testing-rules)）：
  - 测试和开发脚本绝不触碰真实的 `~/.plannotator`、真实 Inbox 或用户配置：在临时 `PLANNOTATOR_DATA_DIR` 下沙箱运行，且只在测试内修改全局变量并恢复。
  - 测试绝不打开浏览器：从 `tests/helpers/no-browser.ts` 设置 `NO_BROWSER`，绝不用 `PLANNOTATOR_BROWSER=none`（或其他哨兵值）。
  - 测试绝不运行真实的 GitHub CLI，也绝不通过 `tests/helpers/inbox-process.ts` 以外的途径向 Inbox 发信号。
  - 每个测试都守护一个可能回归的行为：不做文案快照，不做 Props 往返测试，不回显实现。
  - 在干净环境中测试面向智能体的指令（工具文本、技能、`--help`）：在本仓库之外的临时文件夹中，给一个全新智能体仅提供该文本和一个真实任务。评判它做了什么，而不是它说了什么。
- **PR 检查只覆盖你的改动触及的部分。** PR 只运行其改动文件能触达的测试，而非全套。PR 检查完成后，阅读运行了哪些测试，确认它们覆盖了你的代码。如果没有，自行运行缺失的测试，并在合并前于 PR 中说明。
- **未经所有者批准，不新增 Inbox 键盘快捷键**（见 [键盘快捷键](docs/agent-guides/guides-and-shared-ui.md#keyboard-shortcuts) 中的 Inbox 快捷键说明）。其他位置的快捷键走快捷键注册表。
- **Git：** 绝不对共享分支变基或强制推送；改为合并 main。
- **无 AI 署名。** 提交信息、标签、发布说明和 PR 正文不包含 AI 署名，也不包含会话链接。
- **发布：** 遵循 [`.agents/skills/release/SKILL.md`](.agents/skills/release/SKILL.md)。

## 专题指南

- [`project-and-development.md`](docs/agent-guides/project-and-development.md)：仓库地图、服务端运行时、安装、开发、测试规则、构建、营销站点。
- [`environment.md`](docs/agent-guides/environment.md)：所有环境变量和仅配置项设置。
- [`plan-review.md`](docs/agent-guides/plan-review.md)：计划审查流程、VS Code 中的 Guides、Pi 计划审查和 Pi `plannotator` 工具。
- [`claude-code-mod.md`](docs/agent-guides/claude-code-mod.md)：Claude Code Hook 模块（非阻塞流程、`plannotator` 工具、其 Ask this session 客户端和 Inbox 唤醒、版本偏差）。
- [`opencode-t3-and-codex.md`](docs/agent-guides/opencode-t3-and-codex.md)：OpenCode 2 工具、T3 Code、Codex Stop Hook。
- [`code-review.md`](docs/agent-guides/code-review.md)：代码审查流程和 UI、Ask AI 提供商默认值、"Ask this session"（会话桥和拉取桥）。
- [`annotate.md`](docs/agent-guides/annotate.md)：批注流程、bundle、草稿、严格门禁、实时应用批注。
- [`inbox.md`](docs/agent-guides/inbox.md)：Plannotator Inbox 指南索引（`inbox-*.md`）。
- [`server-and-storage.md`](docs/agent-guides/server-and-storage.md)：归档、Server API 表和守卫、计划历史、反馈归档、计划 diff。
- [`guides-and-shared-ui.md`](docs/agent-guides/guides-and-shared-ui.md)：可移植 Guided Reviews、数据类型、解析器、批注系统、问题块、WebMCP、快捷键、分享、设置、语法高亮。

## 新文档放哪里

保持本文件是地图：不要在此添加功能细节。把新章节放进对应领域的指南（上面的列表）：环境变量放 `environment.md`，端点放 `server-and-storage.md` 中对应的 Server API 表，Inbox 行为放对应的 `inbox-*.md`（新 Inbox 指南还要在 `inbox.md` 中加一行），共享 UI 或格式变更放 `guides-and-shared-ui.md`。新指南在上面的列表中加一行。当本文件超出预算或链接到不存在的指南时，`tests/agents-md-size.test.ts` 会失败。