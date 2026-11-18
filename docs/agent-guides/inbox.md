## Plannotator Inbox

Inbox 的细节按主题拆分。先读基础指南，了解命令、服务器、存储和列表行为，再按你的改动查阅窗口、消息、文件、通知或手机相关链接。

- [`inbox-foundation.md`](inbox-foundation.md) — 命令（`plannotator inbox call`、脚本命令，以及脚本按主机区分）、本地与 tailnet 可达性、注册表、存储、线程和列表。
- [`inbox-window.md`](inbox-window.md) — 主窗口、问题和发送行为。
- [`inbox-messages.md`](inbox-messages.md) — 草稿、回复、线程操作和项目菜单。
- [`inbox-compose.md`](inbox-compose.md) — 撰写消息并将其路由到 agent。
- [`inbox-launch.md`](inbox-launch.md) — 从 New message 启动 Claude Code、Pi 或 Codex（默认开启；Settings → Agents 开关，`inboxLaunch.enabled`）、运行行、Stop、Open 和 Delete。
- [`inbox-routines.md`](inbox-routines.md) — routines：agent 按计划运行的已保存提示词，每次运行都在同一线程中（Settings → Capabilities，`inboxScheduledThreads`，默认关闭）；时钟、错过的运行、agent 的 `routine` 工具、登录时启动、已链接的机器。
- [`inbox-files-and-guides.md`](inbox-files-and-guides.md) — 附件、标注、图片和引导式审查。
- [`inbox-notifications-and-agents.md`](inbox-notifications-and-agents.md) — 搜索、事件流、通知、Board 和 agent 连接。
- [`inbox-phones-and-security.md`](inbox-phones-and-security.md) — 手机配对与界面、安全边界和证明。
- [`inbox-cloudflare-access.md`](inbox-cloudflare-access.md) — 通过用户自己的 Cloudflare Tunnel 和 Access 访问：入口检查、会话路由、设置与拆除。
- [`inbox-machines.md`](inbox-machines.md) — 已链接的机器：跨机器共享一个 Inbox、链接、合并视图、转发和发件箱。
- [`inbox-accounts.md`](inbox-accounts.md) — Settings → Accounts：本机及其已链接机器的 Claude 账户与用量，通过 claude-swap 读取。
- [`inbox-plans.md`](inbox-plans.md) — Plans：计划审查处理过的所有计划的库（config.json `inboxPlans`，默认关闭；`plannotator plans` 可开启）；其记录、磁盘上的库、从 `history/`、`plans/` 和 `feedback/` 的只读导入、能力接缝、`/api/plans` 路由以及窗口中的 Plans 页面。
- [`inbox-system.md`](inbox-system.md) — System：每个项目代码的图（config.json `inboxSystem`，0.29 中隐藏；Settings → Capabilities → System，仅在 Code review 开启时可用）；开关、Reviews System 按钮和项目图、项目设置对话框及其 `setup.json`。