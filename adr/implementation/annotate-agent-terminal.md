# 标注 Agent 终端实现

状态：已在本地实现并验证

## 目标

为 `plannotator annotate` 添加一个可选的、由 WebTUI 驱动的 Agent 终端，用于单文件和文件夹标注。使其与标注导出、Ask AI、代码审查 Agent 任务、计划审查、归档、目标设置以及 `annotate-last` 保持隔离。

## 需保留的决策

- 不自动启动任何 Agent。
- 用户通过左上角的 `ReviewAgentsIcon` 控件启动且仅启动一个选定的 Agent。
- 终端面板与文件侧边栏分离，显示在其左侧。
- 终端在 Plannotator 原始启动 cwd 中启动，而非被标注的文件/文件夹路径（除非两者相同）。
- 终端使用与 Plannotator 相同的 origin 和端口。
- 不新增面向浏览器的 PTY 端口。Bun 在内部使用仅限回环的惰性 Node sidecar，因为 WebTUI 的 PTY 数据通路在 Node 下工作正常。
- 不提供任意命令输入框。
- v1 不包含提示注入、标注同步、文件侧边栏集成、多 Agent、后台 Agent 任务或终端持久化。

## 任务跟踪

- [x] 将 WebTUI 添加为本地开发依赖。
- [x] 添加共享的浏览器安全终端能力类型。
- [x] 添加 Bun annotate 同源 WebSocket 代理、惰性 Node sidecar PTY 支持以及 `/api/plan` 能力。
- [x] 添加 Pi/Node annotate WebSocket/PTY 支持及对应能力。
- [x] 将原始启动 cwd 传入 annotate 服务入口。
- [x] 添加 annotate Agent 终端设置辅助模块。
- [x] 添加独立的 annotate Agent 终端面板。
- [x] 使用 `ReviewAgentsIcon` 添加左上角 Agent 图标/开关。
- [x] 通过 WebTUI 接通启动/停止/重启生命周期。
- [x] 自审服务端适配器的生命周期、禁用回退和同端口行为。
- [x] 自审 UI 的状态隔离和布局约束。
- [x] 验证针对性测试、构建和手动 annotate 流程。

## 备注与发现

- ADR：`adr/0002-add-webtui-agent-panel-for-annotate-mode.md`。
- WebTUI 源码：WebTUI 仓库的本地检出。
- WebTUI 通过 `@plannotator/webtui` 使用；不应重新引入本地文件依赖。
- WebTUI 从 `dist` 导出；修改后需重新构建 WebTUI。
- 当前运行时发现：WebTUI 的 `NodePtyBackend` 在 Bun 下可以创建 PTY，但在本地冒烟测试中未能传递 `onData` 输出。同一后端在 Node 下能正确传递 shell 和 Claude 输出。这与 WebTUI 的可运行示例一致——它们在 Node 中运行 PTY 后端。
- Bun 使用 `packages/server/agent-terminal.ts` 作为同源浏览器 WebSocket 代理。它在首次终端 WebSocket 连接时惰性启动 `packages/server/agent-terminal-node-sidecar.mjs`。annotate 打开时不存在 sidecar 或 PTY。
- Node sidecar 在随机内部端口上绑定 `127.0.0.1`，校验仅启动 WebTUI 内置 Agent，强制使用配置的 Plannotator cwd，并在 Bun 父进程关闭其 stdin 管道时退出。
- Pi/Node 使用 WebTUI 的辅助模块，并将后端包装器挂接到现有 Node HTTP 服务器。
- UI 面板隔离在 `packages/editor/components/AnnotateAgentTerminalPanel.tsx`。
- 停止行为：发送 Ctrl-C，350ms 后再次发送 Ctrl-C，若仍需要则在 1400ms 后杀死 PTY。一次真实的 Codex 冒烟测试确认，当 Agent 未能足够快地从中断中退出时，回退杀死是保证清理的关键。
- 本次提交的当前验证：针对性的 agent-terminal/theme 测试、`bun run --cwd apps/hook build`，以及 annotate-folder 的本地浏览器冒烟测试。
- 实现期间的早期验证覆盖了 Bun 同端口 WebSocket 行为、Pi/Node 同端口 WebSocket 行为、真实 Agent 启动/杀死清理，以及无头 Chrome 布局冒烟测试。

## 后续设计说明

### 主题

Agent 终端外观受三层影响：

- Plannotator 外壳：Tailwind/主题令牌，如 `bg-card`、`border-border` 和 `text-muted-foreground`。
- WebTUI/xterm：`terminalOptions.theme` 和 `terminalColorScheme`。面板在 `packages/editor/components/annotateAgentTerminalTheme.ts` 中将 Plannotator 主题映射到 xterm 调色板。
- 终端内容：用户的 shell 提示符、CLI ANSI 颜色和 Agent UI 颜色。

主题映射是刻意精选的。新的 Plannotator 主题应显式获得终端预设，或回退到从当前 CSS 令牌派生的调色板。

### Agent 消息注入

WebTUI 支持在终端就绪后注入消息：

- `sendAgentMessage({ text })` 使用括号粘贴并提交消息。这是"将此发送给运行中的 Agent"的正确产品 API。
- `write(data)` 发送原始终端输入。除非有特定的高级用例，否则应保持内部使用。
- `pasteText(text)` 可以粘贴但不一定提交。

Plannotator 尚未暴露注入 UI。干净的版本是一个小的"发送给 Agent"入口，挂接到 `WebTuiTerminal` 句柄，并在 `onReady` 触发前禁用。避免暴露原始终端命令框。

### 文件树与编辑器感知

当前状态：

- annotate 文件树不实时监听磁盘变更。
- 文件树从 `/api/reference/files` 获取，仅在 UI 显式请求时刷新。
- main 分支现已支持基于源码的直接编辑。Plannotator 知道通过编辑器打开/保存的文件，并可将树行标记为脏、保存中、已保存、冲突或错误。
- Plannotator 在通过 `/api/source/save` 保存时，通过哈希/mtime 检测保存冲突。
- Plannotator 不会主动察觉 Agent 在编辑器打开期间修改磁盘文件。

推荐形态：

- 为 annotate 根目录添加服务端监听器，忽略现有排除目录。对文件系统突发事件去抖，并通过 SSE 发送紧凑的"树已变更"事件。客户端随后应刷新树快照；不应尝试直接从原始文件系统事件变更树节点。
- 添加独立的基于 git 的工作区变更界面。以 git 作为变更事实来源：`git status --porcelain -z`、`git diff` 和未跟踪文件处理。在树中显示变更文件，并按需惰性打开 diff。
- 保持归因务实。"由 Plannotator 保存"可从源码保存记录得知。"自 Agent 启动后变更"可通过比较启动时快照与当前磁盘/git 状态得知。"Agent 修改了此文件"除非每个写入者都被插桩，否则无法完全确定。
- 对于编辑器，若打开的文件在磁盘上变更且 Plannotator 缓冲区干净，则自动刷新或显示细微的重新加载提示。若缓冲区为脏，则显示冲突提示，包含用户缓冲区与磁盘的 diff，以及磁盘基线与新磁盘的对比。

这样可保持功能可靠且快速：文件系统监听器仅触发失效，git 提供权威的变更文件模型，昂贵的 diff 仅在用户打开时计算。