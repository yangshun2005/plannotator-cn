# 测试

本目录包含 Plannotator 的手动测试脚本。

## 手动浏览器 UI 冒烟测试（`tests/manual/local/`）

这些是仅限本地使用的脚本，用固定数据启动 Plannotator UI 流程，方便你在浏览器中手动验证。它们不是自动化 CI 测试。

**计划审查 UI 冒烟测试：**

```bash
./tests/manual/local/test-hook.sh          # Claude Code simulation
./tests/manual/local/test-hook-2.sh        # OpenCode origin badge test
./tests/manual/local/test-codex-plan-review-e2e.sh  # Real Codex Stop-hook flow in disposable HOME
```

**代码审查 UI：**

```bash
./tests/manual/local/test-opencode-review.sh  # Code review UI test
./tests/manual/local/test-worktree-review.sh  # Worktree support test (creates sandbox with 4 worktrees)
```

详细的 UI 测试文档见 [UI-TESTING.md](../docs/UI-TESTING.md)。

以用户为中心的端到端 Vim 控制矩阵位于
[manual/vim-ux-smoke.md](manual/vim-ux-smoke.md)。它涵盖真实 Markdown 与
原始 HTML 的导航、选择、标注、焦点恢复、HUD 行为以及原生控件兼容性。

## 集成与实用测试（`manual/local/`）

这些脚本测试集成、发布版本，并提供实用工具。

**二进制发布测试：**

```bash
./tests/manual/local/test-binary.sh        # Test installed binary from ~/.local/bin/
```

测试已安装的 `plannotator` 二进制文件，验证发布版本工作正常。

**批量计划测试（Obsidian 集成）：**

```bash
./tests/manual/local/test-bulk-plans.sh    # Iterate through ~/.claude/plans/
```

在 Plannotator 中逐个打开 `~/.claude/plans/` 下的每个 `.md` 文件。适合测试 Obsidian 与多个
计划的集成。

**OpenCode 集成沙箱：**

```bash
./tests/manual/local/sandbox-opencode.sh [--disable-sharing] [--keep] [--no-git]
```

创建一个包含示例 React/TypeScript 项目的临时沙箱，初始化 git 并保留未提交的更改，配置
本地 OpenCode 插件，然后启动 OpenCode 进行完整集成测试。

选项：

- `--disable-sharing`：创建禁用共享的 `opencode.json`
- `--keep`：退出时不清理沙箱
- `--no-git`：跳过 git 初始化（测试非 git 回退）

**Codex Stop-hook 端到端测试工具：**

```bash
./tests/manual/local/test-codex-plan-review-e2e.sh [--keep] [--skip-build]
```

构建 Hook 和审查应用，创建一次性 `HOME` 及示例 git 仓库，复制你的 Codex `auth.json`，
启用 `hooks`，并针对示例项目运行真实的 `codex exec`。脚本会将日志、rollout 路径、
历史索引和会话 URL 写入临时根目录下的产物目录。

提示：

- 若想用 Playwright 驱动打开的计划审查会话而非自动打开浏览器，请设置
  `PLANNOTATOR_BROWSER=/usr/bin/true`。
- 已验证的工作流是：在一个终端运行脚本，然后在另一个终端用 Playwright 指向打印出的
  会话 URL。

**Obsidian 工具：**

```bash
./tests/manual/local/fix-vault-links.sh /path/to/vault/plannotator
```

为仓库中已有的计划文件添加 Obsidian 反向链接（`[[Plannotator Plans]]`）。

## SSH 远程测试（`manual/ssh/`）

测试远程开发场景下的 SSH 会话检测和端口转发。

```bash
cd tests/manual/ssh/
docker-compose up -d
./test-ssh.sh
```

详细设置说明见 [manual/ssh/DOCKER_SSH_TEST.md](manual/ssh/DOCKER_SSH_TEST.md)。