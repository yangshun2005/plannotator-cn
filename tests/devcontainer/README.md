# Plannotator Devcontainer 测试

本目录包含一个 devcontainer 配置，用于在容器化环境中测试 Plannotator 与 OpenCode 的集成。

## 前置条件

1. 已安装并运行 Docker
2. 已安装 VS Code 及 Dev Containers 扩展
3. 宿主机已配置 OpenCode 认证（`~/.local/share/opencode/auth.json`）

## 配置

1. **创建认证软链接**（宿主机一次性配置）：
   ```bash
   mkdir -p .opencode
   ln ~/.local/share/opencode/auth.json .opencode/auth.json
   ```

2. **在 VS Code 中打开**：
   ```bash
   code tests/devcontainer
   ```

3. **在容器中重新打开**：出现提示时点击 "Reopen in Container"，或使用命令面板：`Dev Containers: Reopen in Container`

## 测试 Plannotator

devcontainer 已预配置：
- `PLANNOTATOR_REMOTE=1` — 启用远程模式
- `PLANNOTATOR_PORT=9999` — UI 固定端口
- 端口 9999 转发到宿主机

### 测试步骤

1. 在容器终端中运行 OpenCode：
   ```bash
   opencode
   ```
   或使用 Web 界面：
   ```bash
   opencode web
   ```
   然后在浏览器中访问 http://localhost:4096。

2. 让 OpenCode 创建计划（例如："Create a plan to add user authentication"）

3. 当 OpenCode 调用 `submit_plan` 时，Plannotator 应：
   - 在端口 9999 启动服务器（而非随机端口）
   - 不尝试打开浏览器（远程模式）

4. 在宿主机浏览器中打开 `http://localhost:9999`

5. 批准或拒绝计划

## 预期行为

**修复前（v0.4.0）：** 插件因尝试打开浏览器而挂起，随机端口无法使用

**修复后：**
- 服务器使用固定端口 9999
- 不尝试打开浏览器
- 通过端口转发正常工作

## 故障排查

**插件未更新？**
```bash
rm -rf ~/.cache/opencode/node_modules/@plannotator      # OpenCode 1
rm -rf ~/.cache/opencode/npm/@plannotator/opencode@*    # OpenCode 2
```

**OpenCode 启动时崩溃/中止？**
```bash
rm -rf ~/.cache/opencode
```

**端口未转发？**
检查 VS Code "Ports" 标签页，确认 9999 已列出并转发。

**认证问题？**
确保 `.opencode/auth.json` 存在且包含有效凭据。