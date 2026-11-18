# 自定义审查

自定义审查是一种 Agent Skill。让 Plannotator 指向某个 skill，它就会用该 skill 的指令执行审查。这个 skill 就是审查本身。默认审查指令不会被加入，审查结果照常返回。

## 启用自定义审查

你的全局文件夹（`~/.claude/skills`、`~/.codex/skills`、`~/.agents/skills`）里已经存放了 skill。在 `~/.plannotator/review-skills.json` 中列出你想用作审查的 skill：

```json
{ "version": 1, "enabled": ["security-review", "api-contracts"] }
```

名称即 skill 文件夹名。运行代码审查，选择该 skill，然后执行。审查结果与现在一样返回。

没有 `review-skills.json`？你会得到默认审查，保持不变。

## 工作原理

Plannotator 在启动时读取 skill 的 `SKILL.md` 正文，并将其用作审查提示词。该 skill 定义了审查内容；默认审查提示词被丢弃，用户消息被裁剪为 agent 查找变更所需的 git 或 PR 上下文。读取是实时的。按常规方式编辑 skill，下一次审查就会采用。不会复制任何内容。

有些 skill 带有 `references/`、`scripts/` 或 `assets/`。对于这些，Plannotator 会告诉 agent skill 文件夹的位置，agent 按需从文件原本所在的位置打开它们。

## 什么算作 skill

名称即文件夹名。指令是去掉开头 frontmatter 块后的 `SKILL.md` 正文。Plannotator 不读取 frontmatter：不读 `name`、不读 `description`、不读 YAML。

仅限全局 skill。签入仓库的 skill 会被忽略。（否则来自 Fork 的 Pull Request 可能直接把指令塞进你的审查器。）

## 从 JSON 配置迁移

旧的 `~/.plannotator/reviews/*.json` 配置已移除。迁移方法：把其指令文本放入某个全局 skill 文件夹下的 `SKILL.md`，然后把该 skill 的名称加入 `~/.plannotator/review-skills.json`。