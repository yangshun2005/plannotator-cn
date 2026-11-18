# v0.19.1 — 首次 PR 评审发现的已知问题

跟踪 v0.19 评审的后续事项。以下每节展示阅读器的一处渲染缺陷。有些轻微，有些烦人。均不阻塞 v0.19。

范围：本节每个缺陷都可通过在标注 UI 中阅读本文件复现。

## 1. 智能标点吞掉 CLI 标志

正文中引用命令行标志时，两个连字符会被静默转成短破折号。对比以下两行——源码看起来相同，渲染结果却不同：

- 正文中：运行 bun --watch 以在变更时重新构建，或运行 npm --help 查看选项。
- 代码段中：运行 `bun --watch` 以在变更时重新构建，或运行 `npm --help` 查看选项。

第一行复制粘贴后是损坏的命令。第二行保留字面连字符，因为代码段跳过该转换。

更长的标志同样如此：`claude-code --model opus-4` 在代码中正常，但在正文中——**claude-code --model opus-4**——会渲染成短破折号。

**影响：** 所有在反引号之外提及工具调用的开发计划。

## 2. GitHub 警报正文以列表开头时丢失正文

GitHub 上最常见的标注模式是内含项目符号列表的 Note 或 Warning。目前阅读器会破坏这种结构——警报渲染为空标题，列表项溢出到下方，变成独立的斜体引用。

> [!NOTE]
> - 先安装 bun：`curl -fsSL https://bun.sh/install | bash`
> - 然后克隆并运行：`bun install && bun run dev`
> - 在 #412 中报告问题

> [!WARNING]
> 1. 迁移前停止正在运行的服务器。
> 2. 运行命令前备份 `~/.plannotator/plans`。
> 3. 确认前验证备份。

> [!TIP]
> 正文为纯文本的警报仍正常——被拆分的特指列表 / 代码围栏 / 标题正文。

**影响：** 任何将含真实警报的 GitHub README 段落粘贴到计划中的人。

## 3. 提及和 Issue 引用总是指向 github.com

阅读器根据当前仓库名是否为斜杠形式（org/repo）来链接 `@alice` 和 `#42`。该假设对 GitHub 成立，但对 GitLab、Bitbucket、Azure DevOps 或自托管代码托管平台是错误的——它们都使用相同的 `group/project` 形式。

对于在 GitLab 仓库中使用 Plannotator 的团队，每个 @bob、@carol 或 #123、#456 之类的引用都会生成指向 `github.com/bob` 或 `github.com/group/project/issues/123` 的链接——目标错误。

**影响：** 非 GitHub 团队在每个计划的每处提及上都会看到损坏链接。

## 4. 原始 HTML 中的外部链接劫持评审标签页

Markdown 链接自动在新标签页打开。原始 HTML 块内的链接不会——点击它们会用目标页面替换 Plannotator 标签页，丢失所有进行中的标注。

示例：

<details>
<summary>规范中的参考链接</summary>

参见 <a href="https://tanstack.com/table/v8/docs">TanStack 文档</a> 了解我们沿用的排序/过滤模式，<a href="https://primer.style/design/foundations/tokens">Primer tokens</a> 了解我们对齐警报颜色所用的令牌，以及 <a href="https://github.com/anthropics/claude-code">Claude Code 仓库</a> 了解代理集成细节。

</details>

点击上方任意链接即可复现——整个 Plannotator 会话消失，后退按钮可恢复，但所有未保存状态丢失。

**安全相关提示：** 若粘贴的 HTML 来自不可信来源，这还会打开标签页劫持向量。打开的页面持有指向 Plannotator 标签页的实时引用，可将其重定向。计划随本次一并修复。

## 5. 文本相同的标题共享同一锚点 id

以下两个子节都获得锚点 id `#summary`。点击侧边栏目录中的任一条目，页面总是跳到第一个。片段 URL 如 `plan.html#summary` 同样如此。

### Summary

本节第一版——目录每次都落在这里。

### Summary

本节第二版——无法通过哈希导航到达。

**影响：** 重复结构（每章一个"Summary"、每节一个"Risks"）的较大计划会静默失去侧边栏精度。

## 6. URL 自动链接丢弃末尾右括号

维基百科 URL 常以 `(disambiguation)` 或 `(mathematics)` 结尾。我们的自动链接裁剪末尾标点过于激进，右括号被剥离，留下指向不存在页面的链接。

在 https://en.wikipedia.org/wiki/Function_(mathematics) 阅读相关内容——点击后落到 `Function_(mathematics`，缺少右括号，返回 404。

https://en.wikipedia.org/wiki/Plan_(drawing) 同样如此。包裹后正常：[绘图惯例](https://en.wikipedia.org/wiki/Plan_(drawing))。

**影响：** 任何人链接维基百科或路径中含成对括号的 URL 时。比其他问题少见，但命中时很明显。

## 7. 计划差异视图丢失警报、指令和 details 语义

计划第一版被拒绝、作者重新提交后，评审者切换到差异模式查看变更。目前差异渲染器对 v0.19 的三种新块类型——警报、指令和原始 HTML 块——回退为纯文本，评审者恰在最需要视觉线索的模式下看到无结构内容。

差异模式中被扁平化的示例：

- `> [!WARNING]` 块渲染为普通斜体引用，丢失颜色和图标
- `:::danger` 块完全消失（差异渲染器无匹配分支）
- `<details>` 节渲染为字面 `<details>` / `<summary>` 标签

**影响：** 仅限拒绝 / 重新提交流程。不理想，但差异仍可导航——只是不保留新的语义格式。

## 8. 原始 HTML 中的相对非文档链接未被处理

HTML 块会重写相对图片路径（`<img src="./logo.png">`）和相对 markdown/html 文档链接（`<a href="./notes.md">`），以正确通过服务器路由。其他文件类型未被重写。

<details>
<summary>会损坏的示例链接</summary>

下载规范为 <a href="./docs/spec.pdf">PDF</a>，或获取原始数据为 <a href="./data/sample.csv">CSV</a>。

</details>

两者都相对于 Plannotator 服务器源解析，而非源文件目录，因此即使文件就在计划旁边也返回 404。

**影响：** 少见——用户粘贴含 PDF / 数据 / 图片链接（非 `.md` 或 `.html`）的 README 段落。量小但命中时令人困惑。

## 9. 复制为 Markdown 会损坏单元格含竖线的表格

表格悬停工具栏和弹出窗口都提供"复制为 markdown"。当任一单元格含字面 `|`（常见于列出正则模式、shell 命令或布尔"或"的表格），复制输出会静默出错——竖线未被重新转义，粘贴的表格在这些竖线处多出列。

下方源码（注意 `\|` 转义使表格在此正确解析）：

| Case | Pattern | Notes |
|---|---|---|
| Regex alternation | `cat\|dog` | matches either word |
| Shell pipe | `grep err \| wc -l` | count error lines |
| Boolean | `A \| B` | logical or |

现在悬停该表格，点击复制，粘贴到任意 markdown 文档——"Pattern"列在 `\|` 处被拆成两列，其后每个表头和单元格都错位。表格在阅读器中看起来正常；只在复制时出错。

**影响：** 所有记录正则、shell 或布尔内容的表格。复制时静默损坏——无错误、无警告。

---

## 发布计划

| Issue | 类别 | 目标 | 备注 |
|---|---|---|---|
| 1. CLI 标志智能标点 | 立即修复 | v0.19.1 | 将 `--` 规则收窄为仅数字 |
| 2. 警报 + 列表正文 | 立即修复 | v0.19.1 | 无论块标记如何都合并进警报 |
| 3. 非 GitHub 代码托管平台支持 | 立即修复 | v0.19.1 | 通过 `repoInfo` 传递主机 |
| 4. HTML 外部链接 | 立即修复 | v0.19.1 | 在 `rewriteRelativeRefs` 中强制 `target` + `rel` |
| 5. 重复标题锚点 | 立即修复 | v0.19.1 | 每文档去重计数器 |
| 6. URL 括号裁剪 | 后续 | v0.20 | 成对括号匹配 |
| 7. 差异视图块变体 | 后续 | v0.20 | 扩展 `SimpleBlockRenderer` |
| 8. 相对 HTML 资源链接 | 后续 | v0.20 | 重写任意相对 href，不仅文档 |

准备好后评审。@alice 负责 v0.19.1 范围，@bob 负责 v0.20 待办排序。