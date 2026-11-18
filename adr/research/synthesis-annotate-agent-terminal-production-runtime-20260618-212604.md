# 综合：标注 Agent 终端生产运行时

日期：2026-06-18

状态：综合

## 研究结论

该功能在概念上是正确的。Plannotator 负责标注模式和面向浏览器的 WebSocket。WebTUI 负责终端渲染和 PTY 行为。Node 通过 `node-pty` 负责实际的 PTY。Bun 仍是主服务器运行时。

生产环境的差距很具体：编译后的 Bun 二进制可以将 Node sidecar 文件写入磁盘，但 Node 无法从 Bun 的虚拟文件系统导入 `@plannotator/webtui`。本地开发掩盖了这个问题，因为仓库中存在 `node_modules`。

所以这不是 UI 问题，而是分发/运行时问题。

第二个实际问题是终端 WebSocket。对于一个能在用户工作目录中启动 Agent 的端点来说，静态、未认证的路径过于宽松。

## 应构建的内容

我们应在 Plannotator 安装时添加一个受管理的终端运行时。

该运行时位于 Plannotator 数据目录下，按 WebTUI 版本进行版本管理：

```text
~/.plannotator/vendor/agent-terminal/webtui-0.1.0/
```

它应包含真实的 `node_modules` 树，其中有 `@plannotator/webtui`、`node-pty` 和 `ws`。

编译后的 Plannotator 二进制应基于该运行时解析 Node sidecar。在源码/开发模式下，可以继续使用仓库依赖。

我们还应添加带令牌的 WebSocket 路径：

```text
/api/agent-terminal/pty/<random-session-token>
```

`/api/plan` 应返回带令牌的路径。服务器应拒绝其他所有路径。

## 最佳实现形态

使用一个 Plannotator 自有的运行时安装器路径，然后由平台安装器调用它。

不要在 `install.sh`、`install.ps1` 和 `install.cmd` 中重复 `npm install` 逻辑，而是添加一个内部 CLI 路径：

```text
plannotator install-runtime agent-terminal
```

公共安装器应在安装二进制后调用该命令。如果失败，安装仍然成功，终端功能降级。

这样可将跨平台行为保留在 TypeScript/Bun 代码中（可进行单元测试），同时保持 shell 脚本精简。

运行时安装器应：

- 遵循 `PLANNOTATOR_SKIP_AGENT_TERMINAL_INSTALL=1`
- 要求 Node 20+
- 要求 npm
- 将 `@plannotator/webtui@0.1.0` 安装到受管理的运行时目录
- 避免 audit/funding 噪音
- 如实报告安装失败
- 打印清晰的输出

运行时解析器应：

- 检测源码/开发模式与编译模式
- 在源码/开发模式下使用仓库导入
- 在编译模式下使用受管理的运行时
- 将 `agent-terminal-node-sidecar.mjs` 实体化到受管理的运行时目录
- 在宣告终端已启用之前预检 Node 导入

## 远程模式

远程模式是唯一仍待定的产品决策。

由于远程模式绑定 `0.0.0.0`，最安全的首个发布版本是在远程模式下禁用终端，除非显式启用。

这并不意味着远程用户永远无法使用。这意味着远程终端支持是选择加入的，直到我们有意支持该威胁模型。

建议的环境变量标志：

```text
PLANNOTATOR_AGENT_TERMINAL_REMOTE=1
```

如果远程模式被禁用，标注仍然可用。终端能力会报告为已禁用，并给出明确原因。

## 完成标准

满足以下条件时，该功能达到生产就绪：

- 本地编译安装可以启动终端 sidecar
- 缺少 Node/npm/WebTUI 运行时会产生已禁用的能力，而不是损坏的 Start 按钮
- WebSocket 路径按会话生成且不可猜测
- 远程模式要么被门控，要么被显式支持
- 发布冒烟测试实际测试终端 WebSocket，而不仅仅是 `/api/plan`

## 不在范围内

不要添加任意终端命令。

不要在用户点击 Start 时安装包。

不要重新设计标注 UI。

不要在此生产运行时阶段修复所有终端生命周期边界情况，除非直接涉及。

不要让浏览器负责诊断服务器运行时问题。

## 建议

推进聚焦的实现：

1. 带令牌的终端 WebSocket。
2. 受管理的终端运行时解析器和预检。
3. 内部运行时安装命令。
4. 安装器脚本调用运行时安装命令。
5. 发布冒烟测试证明编译后二进制的终端启动。

这样可保持功能精简，同时消除两个实际发布风险：损坏的编译安装和过于可预测的终端套接字。