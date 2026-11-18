# 审查评论 #7：引用列表回归

本测试夹具隔离了最新 PR 审查中标记的 bug。用**当前**分支状态（修复前）运行它即可看到回归，然后与修复后*应有*的样子对比。

## Bug：引用有序列表塌缩成一行

在此 PR 之前，解析器将每个 `>` 行输出为独立的引用块，因此下面三行渲染为三个堆叠的引用框，分别包含文本 "1. First item" / "2. Second item" / "3. Third item"。不完美但可读。

在此 PR 的引用块合并修复之后，三行合并为单个引用块，内容为 `"1. First item\n2. Second item\n3. Third item"`。渲染器按 `\n\n+`（空行段落分隔）拆分，找不到任何分隔，于是输出**一个** `<p>` 包含整个字符串。在 HTML 中，`\n` 塌缩为空白，因此读者看到一行连排文本：

> 1. First item
> 2. Second item
> 3. Third item

修复后预期：三行视觉上可区分（要么像之前一样堆叠引用框，要么——理想情况下——在一个引用框内呈现正确的嵌套有序列表）。

## 同样的 bug，无序列表版本

> - First bullet
> - Second bullet
> - Third bullet

## 同样的 bug，项目符号 + 文本

> - Install the dependency
> - Run the migration
> - Deploy to staging

## 同样的 bug，内容较长的编号步骤

> 1. Audit the current implementation for any obvious regressions.
> 2. Write a failing test that reproduces the reported issue.
> 3. Land the fix and confirm the test now passes.

## 对照用例：多行折行段落（必须保持修复）

这是引用块合并修复所针对的用例。它应渲染为一个引用框，包含正确折行的散文——而不是六个堆叠的引用框。

> This is a long quoted paragraph that spans multiple source lines because the
> author wrapped it at around 80 columns for readability in the markdown source.
> The renderer should treat this as one continuous paragraph with soft wrapping,
> not six stacked blockquote boxes with individual margins between each line.
> The original bug that motivated the blockquote-merge fix was exactly this
> case — the `tree.hash` quote from the Slice 6 plan. Keep it working.

## 对照用例：多段落引用（必须保持修复）

> First paragraph of a quote that wraps across
> multiple source lines.
>
> Second paragraph of the same quote, separated
> by a blank `>` line. Should render as a visually
> distinct paragraph under the first one.

## 对照用例：两个独立引用（必须保持为两个引用框）

> A quote.

> A completely different quote (blank line between, not `>`-continuation).