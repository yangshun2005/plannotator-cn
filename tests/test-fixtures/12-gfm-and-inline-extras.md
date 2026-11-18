# v0.19 — Markdown 打磨与 GitHub 对齐

让应用内阅读器的渲染效果与 GitHub 对 `README.md` 的渲染保持一致，并改善作者从 PR、聊天和笔记中随意粘贴内容时的行内文本体验。

目标发布：本冲刺末。主要负责人：@backnotprop。评审人：@alice、@bob。

<details>
<summary>本次发布涉及的文件（点击展开）</summary>

| 文件 | 变更 |
|---|---|
| `packages/ui/types.ts` | 新增 `AlertKind`、`'directive'` 联合成员、`alertKind` + `directiveKind` 字段 |
| `packages/ui/utils/parser.ts` | 引用块上的提示检测、指令容器解析 |
| `packages/ui/utils/slugify.ts` | **新增** — `slugifyHeading()` |
| `packages/ui/utils/inlineTransforms.ts` | **新增** — `transformPlainText()` |
| `packages/ui/components/InlineMarkdown.tsx` | 裸 URL 自动链接、Issue 引用、提及、纯文本转换集成 |
| `packages/ui/components/BlockRenderer.tsx` | 标题锚点 id、提示分支、指令分支 |
| `packages/ui/components/blocks/AlertBlock.tsx` | **新增** — 与 GitHub 对齐的提示渲染，使用 Octicons |
| `packages/ui/components/blocks/Callout.tsx` | **新增** — 共享指令容器 |
| `packages/ui/theme.css` | 提示 + 指令颜色变体（浅色 + 深色） |

</details>

---

## 为什么现在做

Plannotator 的阅读器是计划*落地*的地方——计划在这里显示不对，到哪里读都不对。作者经常从 GitHub Issues（#412、#438）、带智能标点的内部文档，以及满是 `:emoji:` 的聊天记录中复制粘贴。目前这些片段渲染出来是直引号、字面 `:wave:` 短代码，以及未链接的 `#123` 引用。体验很粗糙。

> [!NOTE]
> 本次范围仅限**阅读器**。我们不碰写入路径的编辑、草稿持久化或标注存储。这些的后续工作见 #501 和 #512。

我们还从 @carol 那里听说提示样式看起来"像 AI 模板"。说得对——第一版用了大写标题和厚重背景色。本次发布将其对齐到 GitHub 实际的 Primer token。

## 包含内容

八项增量渲染特性，零破坏性变更：

| 特性 | 快捷方式 | 示例 |
|---|---|---|
| 标题锚点 | 自动 | `#why-now` 滚动到这里 |
| 裸 URL 自动链接 | 自动 | https://github.com/backnotprop/plannotator |
| GitHub 提示 | `> [!NOTE]` | 见下文 |
| 指令容器 | `:::note` | 见下文 |
| 提及 | `@user` | @backnotprop |
| Issue 引用 | `#123` | #412 |
| Emoji 短代码 | `:name:` | :rocket: :sparkles: :tada: |
| 智能标点 | 自动 | "it's a feature" → "it's a feature" |

## 详细特性待办

本冲刺所有落地内容、后续工作以及溢出到 v0.20 的工作全貌。展开弹出框以排序/筛选。

| 领域 | 特性 | 负责人 | 评审人 | 优先级 | 状态 | 工作量 | Issue | PR | 目标 | 备注 |
|---|---|---|---|---|---|---|---|---|---|---|
| 阅读器 | 标题锚点 id | @backnotprop | @alice | P0 | 已发布 | S | #541 | #597 | v0.19 | `slugifyHeading`，Unicode 安全 |
| 阅读器 | 裸 URL 自动链接 | @backnotprop | @alice | P0 | 已发布 | S | #542 | #597 | v0.19 | 尾部标点已裁剪 |
| 阅读器 | GitHub 提示 | @backnotprop | @bob | P0 | 已发布 | M | #541 | #597 | v0.19 | Primer token，Octicons |
| 阅读器 | 指令容器 | @backnotprop | @bob | P1 | 已发布 | M | #541 | #597 | v0.19 | 任意类型 |
| 阅读器 | 提及 + Issue 引用 | @backnotprop | @alice | P0 | 已发布 | S | #542 | #597 | v0.19 | 在仓库内时限定 GitHub 范围 |
| 阅读器 | Emoji 短代码 | @backnotprop | @alice | P1 | 已发布 | S | #542 | #597 | v0.19 | 29 个精选代码 |
| 阅读器 | 智能标点 | @backnotprop | @alice | P1 | 已发布 | S | #542 | #597 | v0.19 | 代码段安全 |
| 阅读器 | HTML 块（`<details>`） | @backnotprop | @bob | P0 | 已发布 | L | #489 | #597 | v0.19 | 已净化，相对 URL 重写 |
| 阅读器 | 表格复制（markdown） | @backnotprop | @alice | P1 | 已发布 | S | #562 | #597 | v0.19 | 浮动工具栏 |
| 阅读器 | 表格复制（CSV） | @backnotprop | @alice | P1 | 已发布 | S | #562 | #597 | v0.19 | RFC 4180 转义 |
| 阅读器 | 表格弹出对话框 | @backnotprop | @bob | P1 | 已发布 | M | #563 | #597 | v0.19 | Radix Dialog，标注感知 |
| 阅读器 | 表格排序/筛选 | @alice | @backnotprop | P1 | 进行中 | M | #564 | — | v0.19 | TanStack Table 阶段 B |
| 阅读器 | 表格列可见性 | @alice | @backnotprop | P2 | 计划中 | S | #565 | — | v0.19 | 阶段 C |
| 阅读器 | 表格行选择 | @alice | @backnotprop | P2 | 计划中 | M | #566 | — | v0.19 | 将选中项复制为 CSV |
| 阅读器 | 脚注（`[^1]`） | @bob | @backnotprop | P2 | 计划中 | L | #551 | — | v0.20 | 解析器 + 渲染器 |
| 阅读器 | 数学（`$inline$`） | @bob | @backnotprop | P3 | 待办 | XL | #552 | — | v0.21 | 引入 KaTeX 约 280KB |
| 阅读器 | 任务列表同步 | @carol | @alice | P3 | 待办 | XL | #553 | — | v0.21+ | 双向 Issue 跟踪器同步 |
| 解析器 | 提示大小写不敏感 | @backnotprop | @bob | P1 | 已发布 | S | #541 | #597 | v0.19 | `[!note]` 可用 |
| 解析器 | 指令边界情况 | @backnotprop | @bob | P1 | 已发布 | S | #541 | #597 | v0.19 | 未终止时吸收至 EOF |
| 基础设施 | UI 类型检查接线 | @backnotprop | @alice | P0 | 已发布 | S | — | #597 | v0.19 | 评审发现的根因 |
| 基础设施 | 标注重新水合 | @backnotprop | @alice | P1 | 已发布 | S | — | #597 | v0.19 | `transformPlainText` 回退 |
| 基础设施 | DOMPurify 允许列表 | @backnotprop | @bob | P1 | 已发布 | S | — | #597 | v0.19 | 新增 `open` 属性 |
| UX | 提示样式改版 | @backnotprop | @carol | P0 | 已发布 | M | #538 | #597 | v0.19 | GitHub Primer 对齐 |
| UX | 代码导览对话框 | @rockneurotiko | @backnotprop | P1 | 已发布 | XL | #569 | #569 | v0.18 | 三页动画导览 |
| 文档 | 发布说明草稿 | @backnotprop | @alice | P0 | 进行中 | S | — | — | v0.19 | 亮点 + 近似破坏性变更 |
| 文档 | 博客文章——"阅读器对齐" | @backnotprop | @alice | P1 | 计划中 | M | — | — | v0.19 | 演示视频 + 截图 |
| 文档 | 营销页面更新 | @alice | @backnotprop | P2 | 计划中 | M | — | — | v0.19 | 新特性卡片 |

## 发布计划

:::info
本次发布不设开关——这些特性是增量的，对所有历史计划都能安全渲染。如果任何已保存计划的渲染出问题，那是解析器回归，不是特性开关问题。
:::

三个 PR，按顺序：

1. **提取行内扫描器**（#540）——将 `InlineMarkdown` 从 `Viewer.tsx` 移出，让后续六个特性有干净的落点。纯重构，无行为变更。由 @alice 评审。
2. **块级特性**（#541）——标题锚点、GitHub 提示、指令容器。新增 `slugifyHeading`，检测引用块上的 `[!NOTE]` 标记，解析 `:::kind` 围栏。由 @bob 评审。
3. **行内特性**（#542）——裸 URL 自动链接、提及、Issue 引用、Emoji 短代码、智能标点。新增 `inlineTransforms.ts` 共享工具。由 @alice 评审。

每个 PR 均少于 300 行，并附带测试。新增测试总数：+40。合并顺序很重要——#540 必须先落地，否则 #541 和 #542 会在 `Viewer.tsx` 上冲突。

---

## 特性详解

### 标题锚点

每个标题现在都会根据其文本生成确定性 id，因此 URL 片段可用：`plan.html#rollout-plan` 直接跳转到该章节。行内 markdown（粗体、斜体、代码段）在生成 slug 前被剥离，所以 `**Install** \`bun\`` 变成 `install-bun`，而不是什么奇怪的东西。

Unicode 字母被保留——`Café` 变成 `café`。理由见 #445（一位贡献者在法语计划中的标题产生空 id 后提交了该 Issue）。

### 裸 URL 自动链接

正文中的 URL 现在无需 `<>` 或 `[label](url)` 包裹即可成为链接。将 https://plannotator.ai 粘贴到计划中即可生效。尾部句子标点不会被吞进 URL："Visit https://github.com/backnotprop/plannotator." 句号保持在链接外，与 GitHub 一致。

反引号内的 URL 保持字面——`https://example.com/raw` 显示为代码。显式 `[label](url)` markdown 内的 URL 仍走现有链接处理器。

### GitHub 提示

五种风格，与 Primer 完全一致：

> [!NOTE]
> 用户即使快速浏览也应知道的有用信息。支持 **粗体**、`代码`，以及像 [设计文档](https://plannotator.ai/docs/alerts) 这样的链接。

> [!TIP]
> 有用的建议。试试在 `tests/test-fixtures/12-gfm-and-inline-extras.md` 处用 fixture 运行 `bun run dev:hook`，实时查看整篇文档的渲染。

> [!IMPORTANT]
> 读者必须知道的关键信息。将此 cherry-pick 到补丁版本前请先联系 @backnotprop——#538 中有 PR 描述里没有的上下文。

> [!WARNING]
> 需要注意的事项。如果新增提示类型，请同时更新 `packages/ui/types.ts` 中的 `AlertKind` 和 `AlertBlock.tsx` 中的图标映射。忘记后者会导致静默渲染回退，仅在缺失类型时显现。

> [!CAUTION]
> 负面后果。绝不要在未做共享载荷迁移的情况下发布 `alertKind` 的重命名——旧计划上的标注使用文本搜索进行恢复，因此重命名字段会静默孤立变更前创建的所有提示块标注。迁移手册见 #489。

### 指令容器

与提示同一视觉家族，但支持任意类型。适用于无法干净映射到 GitHub 五种类型的计划约定：

:::tip
指令在项目特定的标注中大放异彩——"部署说明"、"回滚步骤"、"监控检查"。无需将每个概念硬塞进 `note` / `tip` / `warning`。
:::

:::warning
多段指令也可用。

这里的第二段与第一段一起行内渲染。对围栏内的空行免疫。
:::

:::danger
将 `:::danger` 保留给你确实无法撤销的操作——生产数据迁移、破坏性云资源命令、使现有会话失效的凭据轮换。
:::

:::success
在事后复盘和运行手册中使用 `:::success` 作为完成标记。与人们在聊天中已使用的绿色对勾约定一致。
:::

### 提及与 Issue 引用

在正文中写 `@username` 和 `#123`。当计划位于 GitHub 关联仓库内时，它们渲染为指向 github.com 的真实链接。否则渲染为样式化文本——没有坏链接，无需猜测。

- 感谢 @alice 在 #540 中结对完成扫描器提取。
- @bob，你愿意接手解析器 PR（#541）吗？它涉及你最熟悉的 `packages/ui/utils/parser.ts`。
- @carol 提交了 #445、#489，以及 #538 中最初的提示样式反馈。
- 完整讨论历史：#412 → #438 → #489 → #538。

电子邮件地址不会误匹配：ramos@example.com 不是提及。十六进制颜色不会误匹配：`#3b82f6` 仍是颜色，不是 Issue #3。代码段内的文本始终是字面。

### Emoji 短代码

:rocket: 用于发布，:bug: 用于缺陷修复，:sparkles: 用于打磨，:book: 用于文档，:tada: 用于庆祝，:construction: 用于进行中。常用的 GitHub 集合。

正文中行内——"just finished the review :wave:"——按预期渲染。反引号内，短代码保持字面：`:wave:` 显示文本形式。未知短代码原样通过，因此 `:not_a_real_emoji:` 不会静默吞掉你的冒号。

### 智能标点

直引号根据上下文弯曲："she said hello" 变成 "she said hello"。缩略词中的撇号——don't、won't、it's、they'd——都正确弯曲。破折号：两个连字符变成范围用的短破折号（pages 3--5），三个变成长破折号（像这样 --- 行内断句）。

省略号：三个点折叠为正确字符……这意味着作者正文在每个屏幕尺寸上都读起来更紧凑。

代码保持字面。反引号内的 `"don't do this"` 保持直引号，保留代码段内附带的任何 shell 或正则。

---

## 风险与缓解

> [!WARNING]
> 设计期间追踪到的两个真实风险向量：
> 
> **标注文本恢复漂移。** 标注存储从渲染 DOM 捕获的 `originalText`。如果智能标点将 `"hello"` 变成 `"hello"`，本次发布前创建的旧标注在重新加载时找不到自己——它们会静默消失。缓解：文本搜索回退在恢复期间同时规范化直引号和弯引号形式。
> 
> **代码段损坏。** 对 `block.content` 做朴素字符串替换会弯曲 `` `code` `` 段内的引号——错误。缓解：所有行内转换在 `InlineMarkdown` 的纯文本路径内运行，该路径仅在代码段正则已消费代码内容后才到达。

两个风险都不是理论上的——我们在原型阶段都遇到了。具体失败测试用例及其修复见 #541 的 PR 描述。

## 非目标

- **脚注。** 不发布。GitHub 的 `[^1]` 语法是另一个深坑——跟踪于 #551。
- **数学渲染。** `$inline$` 和 `$$block$$` 的 KaTeX 支持——跟踪于 #552。需要引入 `katex`（gzip 后约 280KB），会主导包体积。
- **任务列表元数据同步。** 现有复选框渲染保持不变。与上游 Issue 跟踪器的双向同步是 #553。
- **完整 HTML 透传。** 我们通过现有原始 HTML 块渲染 `<details>` / `<summary>`。任意行内 HTML 仍按设计转义——我们不想重新打开 #489 中净化所解决的 XSS 面。

## 测试

```bash
# 单元测试
bun test packages/ui

# 针对此 fixture 的实时渲染
bun run build:hook && \
  bun run --cwd apps/hook server/index.ts annotate \
  tests/test-fixtures/12-gfm-and-inline-extras.md

# 回归检查：test-fixtures/ 中的每个计划必须仍渲染一致
bun test:regression
```

现有 149 个测试全部继续通过。`slugify.test.ts`、`inlineTransforms.test.ts` 以及 `parser.test.ts` 中引用块提示 / 指令新增部分共 +40 个新测试。

<details>
<summary>预期测试输出（点击展开）</summary>

```
bun test v1.3.11
✓ parser.test.ts — 92 pass
✓ slugify.test.ts — 10 pass
✓ inlineTransforms.test.ts — 9 pass
✓ sanitizeHtml.test.ts — 7 pass
✓ planDiffEngine.test.ts — 14 pass
✓ sharing.test.ts — 11 pass
✓ diagram
```