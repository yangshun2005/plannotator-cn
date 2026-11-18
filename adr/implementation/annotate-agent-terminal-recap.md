# 标注模式的 Agent 终端交接

更新日期：2026-06-18

状态：已在 PR #941 中实现并验证。

PR：https://github.com/backnotprop/plannotator/pull/941

## 概述

Plannotator 标注模式现在为单文件和文件夹标注会话提供了一个可选的、基于 WebTUI 的编码 Agent 终端。

该功能范围保持很窄：

- 仅对 `plannotator annotate <file>` 和 `plannotator annotate <folder>` 生效。
- 不用于计划审查、代码审查、归档、目标设置或 `annotate-last`。
- 仅在用户打开 Agent 面板并点击 Start 后启动。
- 一次只启动一个选定的 WebTUI 内置 Agent。
- 在 Plannotator 启动时所在的目录中启动。
- 在服务端重建实际启动命令；浏览器只选择 Agent 和终端尺寸。
- 渲染为独立可调整大小的面板，位于常规文件/侧边栏区域左侧。
- 允许隐藏面板而不终止正在运行的会话。
- 通过 WebTUI/PTY 生命周期停止：先中断，必要时再终止。

当终端 Agent 就绪时，Ask AI 和 Send Annotations 可以路由到正在运行的终端 Agent。没有就绪的终端 Agent 时，仍可使用现有的 provider/服务端反馈路径。

## 当前运行时形态

浏览器始终与常规的 Plannotator 标注服务端源通信。

面向浏览器的终端 WebSocket 路径现在按标注服务端会话进行令牌化：

```text
/api/agent-terminal/pty/<random-token>
```

静态路径会被拒绝。`/api/plan` 返回该会话唯一有效的 `agentTerminal.wsPath`。

## Bun 运行时

Bun 标注服务端持有面向浏览器的 WebSocket。实际 PTY 运行在惰性启动的 Node sidecar 中，因为 WebTUI 的 `node-pty` 路径在 Node 下更可靠。

流程：

1. 标注服务端启动并解析终端能力。
2. 如果终端支持可用，`/api/plan` 返回 cwd、可用 Agent 和令牌化的 `wsPath`。
3. 浏览器仅在用户启动终端后，才打开令牌化的同源 WebSocket。
4. Bun 校验路径和同主机 `Origin`。
5. Bun 在第一个终端 socket 上惰性启动 Node sidecar。
6. Node sidecar 导入 WebTUI 服务端模块，绑定回环内部 WebSocket，并持有 `node-pty`。
7. Bun 在浏览器 socket 和 sidecar socket 之间代理。
8. Sidecar 校验请求的内置 Agent，并强制使用配置的 Plannotator cwd。

重要文件：

- `packages/server/annotate.ts`
- `packages/server/agent-terminal.ts`
- `packages/server/agent-terminal-runtime.ts`
- `packages/server/agent-terminal-node-sidecar.mjs`
- `packages/shared/agent-terminal.ts`

## 生产运行时

编译后的 Bun 二进制无法让 Node 从 Bun 打包的虚拟文件系统导入 WebTUI。修复方案是在 Plannotator 数据目录下安装一个受管理的磁盘终端运行时：

```text
~/.plannotator/vendor/agent-terminal/webtui-0.1.0/
```

预期内容：

```text
package.json
node_modules/@plannotator/webtui/
node_modules/node-pty/
node_modules/ws/
agent-terminal-node-sidecar.mjs
```

Plannotator 现在有一个内部修复/安装命令：

```bash
plannotator install-runtime agent-terminal
```

该命令：

- 要求 Node 20+ 和 npm
- 安装 `@plannotator/webtui@0.1.0`
- 允许 `node-pty` 安装脚本
- 预检 Node 对 `@plannotator/webtui/core` 和 `@plannotator/webtui/server` 的导入
- 跳过/成功场景以成功退出，真实安装失败以非零退出

安装器在安装二进制后调用此命令：

- `scripts/install.sh`
- `scripts/install.ps1`
- `scripts/install.cmd`

安装脚本将此运行时保持为可选：如果 `plannotator install-runtime agent-terminal` 因 Node/npm 或网络不可用而失败，Plannotator 仍会安装，标注模式在没有集成终端的情况下运行。

退出选项：

```bash
PLANNOTATOR_SKIP_AGENT_TERMINAL_INSTALL=1
```

如果运行时缺失或损坏，标注模式仍会加载，`/api/plan` 报告：

```json
{ "enabled": false, "reason": "runtime-unavailable" }
```

## 远程模式

远程模式将 Plannotator 服务端绑定到 `0.0.0.0`，因此远程会话中终端默认禁用。

需显式启用：

```bash
PLANNOTATOR_AGENT_TERMINAL_REMOTE=1
```

即使启用，Bun 终端桥仍使用令牌化 WebSocket 路径和同主机 `Origin` 校验。

## Pi 运行时

Pi 扩展服务端运行在 Node 上，因此不需要 Bun sidecar 或受管理的编译二进制运行时。

它镜像面向浏览器的契约：

- 由标注服务端提供令牌化 WebSocket 路径
- 相同的 `agentTerminal` 能力形态
- 一次一个终端会话
- 不支持的模式保持禁用

与 Bun 桥不同，当前 Pi/WebTUI 辅助路径不执行单独的 `Origin` 检查。其实际防护是随机路径令牌和相同的远程模式启用门控。

重要文件：

- `apps/pi-extension/server/serverAnnotate.ts`
- `apps/pi-extension/server/agent-terminal.ts`
- `apps/pi-extension/server/agent-terminal.test.ts`

`apps/pi-extension/generated/` 下生成的共享副本被忽略，并通过以下命令重新生成：

```bash
bash apps/pi-extension/vendor.sh
```

## UI 行为

UI 在现有侧边栏控件旁添加了一个左上角 Agent 图标。

面板包括：

- 可用 Agent 选择器
- Start 按钮
- 终端显示的紧凑设置弹出框
- 紧凑的 Stop 控件
- 全出血 WebTUI/xterm 终端表面

冗余的 Agent 标题已移除。运行时，第一行是 Agent 名称、cwd 上下文、设置和停止操作。

终端有意不设圆角卡片边框，也不在 xterm 周围加额外内边距。xterm 滚动条间隙被隐藏，使终端干净地填满面板。

重要文件：

- `packages/editor/App.tsx`
- `packages/editor/components/AnnotateAgentTerminalPanel.tsx`
- `packages/editor/components/annotateAgentTerminalTheme.ts`
- `packages/editor/index.css`
- `packages/ui/utils/annotateAgentTerminal.ts`

## 主题

终端颜色派生自当前活动的 Plannotator CSS 主题令牌。

浏览器尽可能读取实时 CSS 变量，然后将其映射到 xterm/WebTUI 选项。静态主题预设是回退数据，不是主要来源。

这修复了之前灰色/棕色/默认的终端外观，使终端更贴近 Plannotator 的浅色/深色主题。

相关文件：

- `packages/editor/components/annotateAgentTerminalTheme.ts`
- `packages/editor/components/annotateAgentTerminalTheme.test.ts`
- `packages/ui/theme.css`

## Ask AI 和 Send Annotations

当终端 Agent 正在运行且就绪时：

- Ask AI 通过 WebTUI 将问题发送到可见 Agent。
- 基于文件的提问包含活动文件路径，并告知 Agent 从磁盘读取该文件。
- 包含选中/上下文文本，使 Agent 知道用户询问的内容。
- Send Annotations 将导出的标注反馈发送到同一终端 Agent。
- 同一终端会话、目标和反馈正文的重复发送会被阻止。
- 面板内成功发送不会显示额外 toast 噪音。
- 发送到 Agent 后，评论弹出框关闭。

没有就绪的终端 Agent 时，现有 Ask AI 和反馈行为保持不变。

重要文件：

- `packages/editor/App.tsx`
- `packages/editor/agentTerminalIntegration.ts`
- `packages/editor/agentTerminalIntegration.test.ts`
- `packages/ui/components/CommentPopover.tsx`

## 草稿和反馈加固

此工作还包括应随该功能保留的相关清理：

- 标注草稿代数和墓碑，防止过期的自动保存复活已发送的草稿
- 跨运行时共享的集中式反馈模板辅助函数

重要文件：

- `packages/shared/draft.ts`
- `packages/shared/draft.test.ts`
- `packages/ui/hooks/useAnnotationDraft.ts`
- `packages/ui/hooks/useCodeAnnotationDraft.ts`
- `packages/ui/annotationDraftPersistence.test.tsx`
- `packages/shared/feedback-templates.ts`
- `packages/shared/feedback-templates.test.ts`

## 包状态

Plannotator 使用：

```json
"@plannotator/webtui": "^0.1.0"
```

该包发布为 `@plannotator/webtui@0.1.0`。

React 和 ReactDOM 仍是 WebTUI 的 peer 依赖。Plannotator 持有 React 运行时，避免重复 React 问题。

不要重新引入 WebTUI 的本地 `file:` 依赖。

## ADR

先前已接受的 ADR：`adr/0002-add-webtui-agent-panel-for-annotate-mode.md`。

## 验证

当前已在本地完成验证：

```bash
bun test packages/shared/agent-terminal.test.ts packages/server/agent-terminal.test.ts apps/pi-extension/server/agent-terminal.test.ts scripts/install.test.ts
bun run typecheck
bun run build:hook
bun run build:pi
git diff --check
```

额外冒烟检查已完成：

- `PLANNOTATOR_SKIP_AGENT_TERMINAL_INSTALL=1 bun apps/hook/server/index.ts install-runtime agent-terminal`
- 将 `install-runtime agent-terminal` 实际安装到临时 `PLANNOTATOR_DATA_DIR`
- 编译后的 Bun 二进制在已安装受管理运行时下的冒烟测试：
  - 安装运行时
  - 启动 `plannotator annotate README.md`
  - 获取 `/api/plan`
  - 连接到返回的令牌化终端 WebSocket
  - 发送不带 Agent 的 spawn 请求
  - 确认 sidecar 返回 WebTUI 预期的协议错误
- 编译后的 Bun 二进制在没有受管理运行时下的冒烟测试：
  - 启动 `plannotator annotate README.md`
  - 确认 `/api/plan` 报告 `runtime-unavailable`
- 发布工作流 YAML 解析成功

本地未安装 `actionlint`，因此未运行 GitHub Actions 专用 linter。

## 发布工作流

`.github/workflows/release.yml` 现在在 Linux 和 Windows 上对编译二进制终端路径进行冒烟测试。

冒烟测试准备临时受管理运行时，启动标注，获取 `/api/plan`，打开返回的令牌化 WebSocket，并验证 Node sidecar 通过 WebTUI 协议响应。

这很重要，因为之前的发布冒烟只证明 `/api/plan` 加载。它没有证明 Node 能从编译安装中导入 WebTUI。

## 已知非目标

此功能不添加：

- 任意终端命令输入
- 多个同时运行的终端 Agent
- 终端持久化或保存的滚动回看
- 文件树实时监视
- git 变更文件徽章
- 磁盘编辑的 Agent 归属
- 计划审查或代码审查终端支持
- `annotate-last` 终端支持

## 后续工作

有用的后续工作：

- 运行 `actionlint` 或 CI 以验证编辑后的发布工作流
- 决定是否将 ADR 草稿提升为已接受的正式 ADR
- 为标注文件夹添加实时文件树失效
- 添加 git 支持的变更文件展示和惰性 diff
- 为 Agent 修改的文件添加编辑器磁盘变更感知
- 如果安装体积重要，考虑后续提供更小的仅服务端 WebTUI 包
- 如果 `node-pty` 安装摩擦在现场出现，考虑预构建的平台特定终端运行时归档