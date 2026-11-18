# 使用 Docker 测试 SSH 远程支持

此配置创建一个带 SSH 服务器的 Docker 容器，用于测试 Plannotator 的 SSH 远程会话检测。

## 构建并运行

```bash
# 从仓库根目录进入此目录
cd tests/manual/ssh

# 构建 Docker 镜像
docker-compose build

# 启动 SSH 服务器
docker-compose up -d

# 检查是否正在运行
docker-compose ps
```

## 测试 SSH 检测

### 方式 1：SSH 进入容器并运行测试脚本

```bash
# SSH 进入容器（密码：testpass）
ssh -p 2222 root@localhost

# 进入后运行测试脚本
cd /app
chmod +x test-ssh.sh
./test-ssh.sh
```

应看到：
- `[SSH Remote Session Detected]` 消息
- SSH 端口转发说明
- 服务器运行在端口 19432

### 方式 2：通过 SSH 端口转发测试

```bash
# 在一个终端中，使用端口转发 SSH 连接
ssh -p 2222 -L 19432:localhost:19432 root@localhost

# 在 SSH 会话中运行：
cd /app
echo '{"tool_input":{"plan":"# Test Plan\n\nTest content"}}' | bun run apps/hook/server/index.ts

# 在本地机器的另一个终端中打开浏览器
open http://localhost:19432
```

### 方式 3：测试本地（非 SSH）模式

```bash
# 不通过 SSH，直接在容器中执行
docker-compose exec plannotator-ssh bash -c 'cd /app && echo "{\"tool_input\":{\"plan\":\"# Test Plan\n\nTest content\"}}" | bun run apps/hook/server/index.ts'
```

应看到：
- 没有 `[SSH Remote Session Detected]` 消息
- 随机分配端口（因为未设置 SSH_TTY 和 SSH_CONNECTION）

## 验证 SSH 环境变量

```bash
# SSH 进入容器
ssh -p 2222 root@localhost

# 检查 SSH 环境变量是否已设置
echo "SSH_TTY: $SSH_TTY"
echo "SSH_CONNECTION: $SSH_CONNECTION"
```

## 清理

```bash
docker-compose down
```

## 环境变量覆盖

测试自定义端口：

```bash
ssh -p 2222 root@localhost
cd /app
PLANNOTATOR_PORT=9999 ./test-ssh.sh
```

服务器应使用端口 9999 而非 19432。