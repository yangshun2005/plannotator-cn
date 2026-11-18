# Spike：标注 Agent 终端生产运行时

日期：2026-06-18

状态：研究

## 目标

确定 Plannotator 在将标注模式的 WebTUI 终端发布给大量用户之前还需要什么。

该功能在本地开发中已可用：标注模式能显示 WebTUI 终端、启动一个选定的编码 Agent、将 Ask AI / Send Annotations 路由到该运行中的 Agent，并按 Plannotator 当前主题为终端设置主题。

剩下的问题是生产可靠性：编译后的 Bun CLI、Node sidecar、WebTUI 包、WebSocket 路由、安装器和发布冒烟测试应如何配合。

## 当前运行时形态

浏览器通过 `@plannotator/webtui/react` 渲染终端。

Bun 标注服务器持有面向浏览器的端点。在 `packages/server/annotate.ts` 中，`/api/plan` 返回 `agentTerminal` 能力元数据，`/api/agent-terminal/pty` 升级为 WebSocket。

Bun WebSocket 桥接位于 `packages/server/agent-terminal.ts`。

实际 PTY 运行在 Node 中，而非 Bun。Bun 启动 `packages/server/agent-terminal-node-sidecar.mjs`，该 sidecar 导入 `@plannotator/webtui/core` 和 `@plannotator/webtui/server`。

sidecar 绑定到 `127.0.0.1` 上的随机内部端口。浏览器看不到该端口，只连接 Plannotator 正常的标注服务器端口。

Pi 运行时是独立的 Node 服务器代码。它应保持相同的面向浏览器能力形态和路由语义。

## 已验证的生产缺口

编译后的 Bun 二进制可以将 `agent-terminal-node-sidecar.mjs` 实体化到 Plannotator 数据目录下的真实文件。

这还不够。Node 仍须从磁盘上真实的 `node_modules` 导入 `@plannotator/webtui`。

本地开发中这可行，因为仓库的 `node_modules` 存在。

在编译后的 Bun 二进制中，`import.meta.resolve("@plannotator/webtui/core")` 可能指向 Bun 打包的虚拟文件系统。Node 无法从该虚拟文件系统导入包。

实际失败模式很糟：`/api/plan` 可能宣称终端已启用，但点击 Start 会失败，因为 Node sidecar 在报告就绪前就退出了。

所以发布阻塞点不是 React 面板，而是生产终端运行时。

## 仓库中的现有模式

最接近的现有模式是语义 diff sidecar：

- 安装脚本将受管可选运行时放到 `~/.plannotator/vendor/sem/<version>/`。
- 运行时优先使用受管二进制（若存在）。
- 安装失败不致命。Plannotator 仍会安装，可选功能降级。
- 安装有界且显式，不是正常使用应用时的意外网络操作。

编译后的二进制在 `packages/server/codex-review.ts` 中已有一个较小的实体化模式：schema JSON 嵌入 Bun 并写入真实文件，因为外部进程无法读取 Bun 虚拟路径。

目前没有在首次启动终端时执行 `npm install` 的模式。这很好。我们不应为此功能创建这种模式。

## 推荐设计

使用安装器管理的终端运行时。

建议位置：

```text
~/.plannotator/vendor/agent-terminal/webtui-0.1.0/
```

内容：

```text
agent-terminal-node-sidecar.mjs
package.json
node_modules/
```

安装的 `node_modules` 必须包含 `@plannotator/webtui@0.1.0` 及其服务端依赖，尤其是 `node-pty` 和 `ws`。

安装脚本应在 Plannotator 安装期间、主二进制就位后创建此运行时。应遵循语义 diff sidecar 风格：

- 跳过环境变量，例如 `PLANNOTATOR_SKIP_AGENT_TERMINAL_INSTALL=1`
- Node 版本检查，因为 WebTUI 要求 Node 20+
- 失败不致命
- 清晰的安装输出
- 版本化安装目录
- 浏览器点击路径中不做运行时网络安装

运行时行为应诚实：

- 源码/开发模式，使用仓库依赖路径。
- 编译模式，使用数据目录下的受管运行时。
- 若受管运行时缺失或损坏，`/api/plan` 应报告 `agentTerminal.enabled: false` 并给出有用信息。
- UI 不应为无法启动的终端显示 Start 按钮。

Node sidecar 应从受管运行时目录运行，或接收指向该目录的文件 URL。仅当 Node 模块解析锚定在受管运行时中时，裸导入才可接受。

## WebSocket 保护

当前终端 WebSocket 路径是静态的：

```text
/api/agent-terminal/pty
```

对该端点来说太弱，因为该端点可在用户工作目录中启动由终端支持的编码 Agent。

在路径中使用随机会话令牌：

```text
/api/agent-terminal/pty/<session-token>
```

服务器应在标注服务器启动时生成令牌，从 `/api/plan` 返回带令牌的 `wsPath`，并拒绝不匹配的升级请求。

这不是“企业级认证”。只是简单的同会话密钥，使可预测端点不足以驱动终端。

在运行时暴露足够信息时，还应验证请求来源主机。

远程模式需要明确的产品决策。由于远程模式绑定 `0.0.0.0`，最安全的首个发布方案是：

- 默认在远程模式禁用终端，通过环境变量/配置标志选择启用，或
- 仅在带令牌路径和来源检查就位后才允许远程模式。

为广泛发布，保守路径是默认禁用远程，直到我们有意支持该威胁模型。

## 需要的发布验证

添加命中真实失败点的测试：

1. 能力生成的单元测试：启用、禁用、运行时缺失、带令牌路径。
2. 源码模式下 Bun 服务器 WebSocket 冒烟测试。
3. 编译二进制冒烟测试：启动 `plannotator annotate README.md`，获取 `/api/plan`，打开返回的 `agentTerminal.wsPath`，发送 spawn 请求，期望真实协议响应而非 sidecar 导入失败。
4. shell、PowerShell 和 cmd 安装器的安装器测试，确保受管终端运行时不会从发布流程中静默消失。

当前发布冒烟只检查 `/api/plan` 有响应。这不能证明 sidecar 能导入 WebTUI 或启动 PTY。

## 不要做什么

不要在首次点击 Start 时安装 `@plannotator/webtui`。这会造成缓慢、依赖网络、难以调试的用户交互。

除非服务器有足够证据表明运行时能实际启动，否则不要宣称终端已启用。

不要将其打包为接受任意命令的原始浏览器终端。发布面应保持为“在 Plannotator 启动目录中启动一个选定的 WebTUI 内置 Agent”。

不要让 UI 承担生产运行时错误。服务器应暴露诚实的能力状态。

## 待定决策

首个公开版本的远程终端支持：

- 首选：远程模式默认关闭。
- 备选：带令牌 socket 和来源验证后开启。

安装器实现深度：

- 第一版：npm 管理的运行时目录。
- 后续改进：若 `node-pty` 安装摩擦太大，使用预构建的平台特定终端运行时归档。

包拆分：

- 当前 `@plannotator/webtui` 包可用。
- 未来的 `@plannotator/webtui-server` 包可减少安装体积，但安全发布此功能并不需要。

## 建议实现顺序

1. 添加带令牌的 WebSocket 路径和远程模式门控。
2. 添加终端运行时解析器和预检。让 `/api/plan` 诚实。
3. 按语义 diff sidecar 模式，向 `install.sh`、`install.ps1` 和 `install.cmd` 添加安装器管理的终端运行时。
4. 为编译后的标注终端启动添加发布冒烟覆盖。
5. 生产路径验证后，处理较小的 UI 生命周期边界情况。

## 结论

该功能的架构是合理的：Bun 拥有 Plannotator，Node 拥有 PTY，WebTUI 拥有终端行为。

生产发布缺少一层：Node 可从编译后的 Plannotator 安装中导入的受管磁盘终端运行时。

一旦存在，服务器就能可靠地宣称该功能，UI 可保持简单，失败会变成正常的可选功能降级，而不是损坏的 Start 按钮。