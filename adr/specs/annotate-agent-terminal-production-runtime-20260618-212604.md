# 规格：标注 Agent 终端生产运行时

日期：2026-06-18

状态：草案

## 意图

让标注模式下的 WebTUI 终端在编译安装的生产环境中安全可靠。

用户体验保持简单：终端可用时，用户可从标注模式启动一个选定的 WebTUI 内置 agent；不可用时，标注模式仍正常工作，UI 不提供损坏的启动流程。

## 用户可见行为

本地非远程会话中，服务器报告终端能力时，标注模式显示 agent 按钮。

终端仅在用户选择启动后才启动。

生产运行时缺失或损坏时，终端功能禁用，文档标注 UI 仍正常加载。

远程会话默认不启用终端。用户可通过以下方式启用：

```text
PLANNOTATOR_AGENT_TERMINAL_REMOTE=1
```

远程终端支持禁用时，UI 应将其视为其他不可用能力。

## 运行时布局

托管运行时位置：

```text
~/.plannotator/vendor/agent-terminal/webtui-0.1.0/
```

安装/运行时物化后的预期内容：

```text
package.json
node_modules/@plannotator/webtui/
node_modules/node-pty/
node_modules/ws/
agent-terminal-node-sidecar.mjs
```

sidecar 文件可由编译后的 Plannotator 二进制在运行时写入。npm 依赖必须由安装器路径安装，不能由浏览器启动操作安装。

## 新增或更新的环境变量

```text
PLANNOTATOR_SKIP_AGENT_TERMINAL_INSTALL=1
```

在 Plannotator 安装期间跳过托管终端运行时安装。

```text
PLANNOTATOR_AGENT_TERMINAL_REMOTE=1
```

在带令牌的 WebSocket 保护就位后，启用远程模式下的终端。

## 共享类型

更新 `packages/shared/agent-terminal.ts`。

保持此文件浏览器安全。

添加基础路径和辅助函数：

```ts
export const AGENT_TERMINAL_WS_BASE_PATH = "/api/agent-terminal/pty";
export function buildAgentTerminalWsPath(token: string): string;
```

保留 `AgentTerminalCapability.wsPath` 为 `/api/plan` 返回的浏览器可见路径。

添加禁用原因：

```ts
| "remote-disabled"
| "runtime-unavailable"
```

不要在此浏览器共享文件中放入 Node 文件系统或进程逻辑。

## 服务器运行时解析器

添加 `packages/server/agent-terminal-runtime.ts`。

职责：

- 定义 `AGENT_TERMINAL_WEBTUI_VERSION`
- 从 `getPlannotatorDataDir()` 计算托管运行时目录
- 判断源码/开发模式与编译模式
- 查找 `node`
- 校验 Node 20+
- 在编译模式下校验托管 WebTUI 运行时
- 将 `agent-terminal-node-sidecar.mjs` 物化到运行时目录
- 在 `/api/plan` 宣告终端支持启用前，用 Node 预检 sidecar 导入
- 为内部安装命令安装托管运行时

预检不应启动 agent 或分配 PTY。

建议的预检命令：

```text
node --input-type=module -e "await import('@plannotator/webtui/core'); await import('@plannotator/webtui/server')"
```

运行时将 `cwd` 设为托管运行时目录，以便 Node 解析托管 `node_modules`。

使用短超时，失败时返回禁用的能力。

## Bun 桥接

更新 `packages/server/agent-terminal.ts`。

行为：

- 为每个标注服务器会话生成随机令牌
- 将能力 `wsPath` 设为带令牌的路径
- 暴露路径匹配器或在 `upgrade` 内校验
- 拒绝静态 `/api/agent-terminal/pty`
- 通过 `PLANNOTATOR_AGENT_WS_PATH` 将带令牌的路径传给 Node sidecar
- 以锚定到已解析运行时的 `cwd` 或导入 URL 启动 Node
- 浏览器打开终端 socket 时保持 sidecar 懒启动
- 保持一次只有一个活动终端会话

Bun 桥接来源处理：

- 存在 `Origin` 头时，要求其匹配请求主机
- 允许测试和非浏览器客户端缺少 origin
- 远程模式单独门控

## 标注服务器

更新 `packages/server/annotate.ts`。

不要将请求路径与共享静态 WebSocket 路径比较。

改为委托给桥接：

```ts
if (agentTerminal.matches(url.pathname)) {
  ...
}
```

`/api/plan` 应原样返回已解析的桥接能力。

## Pi 运行时

更新 `apps/pi-extension/server/agent-terminal.ts` 和 `apps/pi-extension/server/serverAnnotate.ts`。

Pi 不需要编译后的 Bun 托管运行时，但应镜像相同的浏览器可见语义：

- 带令牌的 WebSocket 路径
- 相同的能力结构
- 不支持的标注模式使用相同的禁用原因
- 一次一个会话

当前 Pi 实现使用 WebTUI 的 Node WebSocket 辅助函数，该函数不暴露单独的 `Origin` 校验钩子。除非 WebTUI 添加该钩子或 Plannotator 替换该辅助函数，Pi 依赖随机路径令牌和相同的远程模式启用门控。

若共享生成文件受影响，更新 `apps/pi-extension/vendor.sh`。

## 内部运行时安装命令

在 `apps/hook/server/index.ts` 中添加内部 CLI 路径：

```text
plannotator install-runtime agent-terminal
```

此命令应：

- 调用服务器运行时安装器
- 打印清晰的成功、跳过或失败消息
- 跳过/成功时退出 `0`，真实安装失败时退出非零
- 让安装脚本决定终端运行时失败对整体 Plannotator 安装非致命

安装器实现应在运行时目录创建最小包并运行 npm：

```text
npm install --omit=dev --no-audit --no-fund @plannotator/webtui@0.1.0
```

不要使用 `--ignore-scripts`；`node-pty` 可能需要安装脚本。

## 安装脚本

更新：

- `scripts/install.sh`
- `scripts/install.ps1`
- `scripts/install.cmd`

二进制安装后，调用：

```text
plannotator install-runtime agent-terminal
```

该调用非致命。若 Node/npm/包安装失败，打印清晰的跳过消息并继续。

遵循：

```text
PLANNOTATOR_SKIP_AGENT_TERMINAL_INSTALL=1
```

保持语义 diff sidecar 逻辑不变。

## 发布工作流

更新 `.github/workflows/release.yml`。

编译二进制冒烟测试必须覆盖终端运行时路径：

1. 在临时 `PLANNOTATOR_DATA_DIR` 中安装或准备托管终端运行时
2. 启动编译后的 `plannotator annotate README.md`
3. 获取 `/api/plan`
4. 断言 `agentTerminal.enabled === true`
5. 打开 `ws://...${agentTerminal.wsPath}`
6. 发送不带 agent 的 spawn 请求
7. 断言响应是预期的 WebTUI 协议错误，而非 sidecar 导入失败

这证明 Node 可从编译后的 Plannotator 二进制导入托管 WebTUI 运行时。

## 测试

添加或更新聚焦测试：

- `packages/shared/agent-terminal.test.ts`
  - 带令牌路径辅助函数
  - 支持的标注模式仍不变

- `packages/server/agent-terminal.test.ts`
  - 功能关闭时禁用
  - 远程模式开启但未启用时禁用
  - 启用的能力使用带令牌路径
  - 不接受静态路径
  - 运行时缺失报告 `runtime-unavailable`

- `apps/pi-extension/server/agent-terminal.test.ts`
  - 带令牌同端口 WebSocket 往返
  - 拒绝静态路径
  - annotate-last 保持禁用

- `scripts/install.test.ts`
  - 所有安装器提及 `PLANNOTATOR_SKIP_AGENT_TERMINAL_INSTALL`
  - 所有安装器调用 `install-runtime agent-terminal`
  - 调用非致命

- 发布冒烟
  - 编译二进制终端 WebSocket 成功到达 sidecar

## 非目标

不添加任意命令入口。

不添加多个终端会话。

不从浏览器启动操作安装 npm 包。

不重新设计终端面板。

不将此功能扩展到计划审查、代码审查、归档或 `annotate-last`。

本轮不解决文件监听、git diff 呈现或编辑器感知。

## 验收标准

源码模式：

- 标注仍可启动
- WebTUI 安装后终端可启动
- 带令牌 WebSocket 可用
- Ask AI 和 Send Annotations 仍路由到就绪终端

编译二进制模式：

- 安装命令可准备托管运行时
- `/api/plan` 仅在运行时预检通过时宣告终端启用
- 终端 WebSocket 到达 Node sidecar
- 运行时缺失禁用终端但不破坏标注

远程模式：

- 终端默认禁用
- 启用后走正常的带令牌路径流程

发布 CI：

- sidecar 无法导入 WebTUI 时二进制冒烟失败

## 实施顺序

1. 添加带令牌路径辅助函数和桥接路径校验。
2. 添加远程模式门控。
3. 添加运行时解析器和预检。
4. 添加内部安装命令。
5. 接线安装脚本调用该命令。
6. 添加测试。
7. 添加编译二进制终端冒烟。
8. 运行类型检查、聚焦测试、构建和冒烟。