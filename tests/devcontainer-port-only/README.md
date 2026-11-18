# Plannotator 测试 - 仅端口（预期失败）

本测试模拟用户在 Docker 中运行 `opencode web` 时报告的常见错误配置：只设置 `PLANNOTATOR_PORT` 而未设置 `PLANNOTATOR_REMOTE`。

## 设置

在 devcontainer 中打开之前，先从宿主机创建认证符号链接：

```bash
mkdir -p .opencode
ln ~/.local/share/opencode/auth.json .opencode/auth.json
```

## 问题

Docker/devcontainer 环境中的用户常设置：
```bash
PLANNOTATOR_PORT=9999
```

但忘记设置：
```bash
PLANNOTATOR_REMOTE=1
```

缺少 `PLANNOTATOR_REMOTE=1`（且环境中无 `SSH_TTY`/`SSH_CONNECTION`）时，插件会：
1. ✅ 使用端口 9999
2. ❌ 仍尝试打开浏览器（静默失败或挂起）

## 预期行为

在此 devcontainer 中触发计划时：
- 服务器在端口 9999 启动
- 插件尝试打开浏览器（失败）
- 用户无任何反馈
- 表现为挂起

## 修复方法

用户需要同时设置两个环境变量：
```bash
PLANNOTATOR_REMOTE=1
PLANNOTATOR_PORT=9999
```

正确配置参见 `tests/devcontainer/`。

## 测试

1. 在 VS Code 中打开此文件夹
2. 在容器中重新打开
3. 运行 `opencode web` 启动 Web 界面
4. 通过转发端口（通常为 4096）访问 OpenCode
5. 请求一个计划
6. 观察挂起/失败——端口 9999 上无任何响应
7. 与正常工作的 `tests/devcontainer/` 对比