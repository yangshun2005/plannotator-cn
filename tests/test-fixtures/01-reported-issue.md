## Web 研究子代理

### 变更

**新文件：`home/dot_config/opencode/exact_agents/web-researcher.md`**

子代理定义，包含：
- 模型：kimi-k2p5-turbo（与其他子代理一致）
- 模式：subagent（不隐藏；也可由用户直接调用）
- 权限：默认全部拒绝，仅允许 `web *` 和 `jq *`（bash），以及 Context7 MCP 工具
- 提示词涵盖：
  - 回退优先级：Context7（精选库文档）→ Web 搜索 → Web 抓取
  - 错误上报：必须无条件、原样向调用方报告所有工具失败（认证错误、超时、HTTP 错误、空结果）
  - 渐进式细化：若初始结果太少，细化查询后再次搜索
  - 最大迭代次数限制，防止失控循环
  - “卡住时”的指导
  - 输出模板（强制结构以保证质量）：