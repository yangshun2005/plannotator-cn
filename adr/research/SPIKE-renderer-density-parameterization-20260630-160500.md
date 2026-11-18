# SPIKE：让 Markdown 渲染器的尺寸/间距可参数化

日期：2026-06-30

## 问题

块渲染器的文字尺寸和间距是按整页方案写死的（h1 24px、正文 15px、大边距）。PR 面板需要更小、更紧凑。让渲染器密度变成可调参数的最干净做法是什么，最好复用代码库已有的模式？

## 发现：尺寸分散各处，没有单一开关

写死的尺寸/间距以**内联 Tailwind 工具类**形式散落在约 9 个组件中，而非集中一处：

- `BlockRenderer.tsx` — `text-2xl/xl/base/[15px]/sm`、`mb-4 mt-6 my-4 mb-2`、`my-8`（hr）。
- `blocks/CodeBlock`（`text-[13px]`、`my-5`）、`blocks/TableBlock`（`text-sm`、`my-4`、`py-2 px-3`）、`blocks/HtmlBlock`（`text-[15px]`、`my-4`）、`blocks/Callout` + `blocks/AlertBlock` + `blocks/proseBody`（`text-[15px]`、`my-4`、`mt-2`）、`InlineMarkdown`（`text-sm`、行内代码尺寸）、`ListItemBody`（`text-sm`）。

所以目前**无法**缩放渲染器——每个尺寸都是元素上的字面工具类。任何参数化都必须要么 (a) 把这些字面值换成可配置的东西，要么 (b) 从外层作用域覆盖它们。

## 代码库已有可复用的模式

1. **CSS 变量字号阶梯** — `theme.css:107-110`：`--text-sm: 0.875rem; --text-sm--line-height: 1.45;` … 直到 `--text-xl`。Tailwind v4 会读取这些变量。渲染器在部分地方忽略了它（用任意值 `text-[15px]`、固定 `text-2xl`），但阶梯是存在的。
2. **变量驱动的字号覆盖，已用于此目的** — `--diff-font-override` / `--diff-font-size-override`（`review-editor/index.css:282, 1275+`）。在 `:root` 上设置该变量，会通过作用域选择器级联进 diff/评论 Markdown，覆盖内联尺寸。这是"用 CSS 变量 + 作用域驱动尺寸"的可行先例。
3. **`.compact` 修饰符先例** — `.suggestion-block.compact`（`index.css:869`）。
4. **`configStore` / `useConfigValue`** 已承载用户字体设置（正是它供给 diff 覆盖）。

## 方案（含权衡）

**A. CSS 变量尺寸/节奏层（推荐）。**
引入一小组 `--md-*` 变量用于字号阶梯 + 块节奏（如 `--md-h1`、`--md-h2`、`--md-body`、`--md-block-gap`、`--md-code`），让渲染器元素读取它们，默认值设为当前方案值（这样方案查看器视觉不变），并新增 `.md-compact` 作用域为 PR 面板设置更小的值。
- *动态性：* 密度成为真正的开关——现在紧凑/默认，日后可轻松让用户调节（可挂在 `configStore` 上，就像 diff 字号那样）。
- *成本：* 约 9 个组件必须读取变量而非字面工具类（`text-2xl` → 变量支撑的类/`style`，`mb-4` → `var(--md-block-gap)`）。一次性改动，但涉及方案查看器共用的组件——需把默认值钉在当前值上，保证方案像素级一致。
- *契合先例：* 与 `--diff-font-size-override` 和 `--text-*` 阶梯同形。

**A-lite. 纯 CSS 覆盖作用域（改动最少）。**
不动组件；在 CSS 中加 `.md-compact` 作用域，用元素选择器（`.md-compact h1 { font-size:…; margin:… }`），靠特异性（元素+类 > 类）压过内联工具类，正如 `--diff-font-size-override` 的作用域选择器今天的做法。
- *优点：* 零组件改动；隔离在一个 CSS 块；不会让方案查看器回归。
- *缺点：* 脆弱——必须镜像组件设置的每个尺寸/边距，组件类一改就会悄悄漂移。一份"影子样式表"。

**B. 将 `density`/`size` prop 逐层穿透 BlockRenderer + 每个块组件。**
- *优点：* 显式、类型安全。
- *缺点：* 侵入性最强——每个组件都要加 prop 和密度→类映射。冗长，且 prop 还得穿透 `groupBlocks`/Viewer。最不优雅。

**C. Tailwind `prose` + `prose-sm`。**
- 意味着把渲染器重写到排版插件上（丢弃逐元素工具类）。大重写，且这里没有任何东西是这样渲染的。否决。

## 建议（供规格）

采用 **A——`--md-*` CSS 变量层**，默认设为当前方案值，并在 PR 面板上用 `.md-compact` 作用域覆盖。这是唯一既*干净*（字号阶梯 + 节奏的单一事实来源）又*动态*（现在紧凑，日后经 `configStore` 可让用户调节，就像 diff 字号）的方案。它直接复用两个现有模式（`--text-*` 阶梯、`--diff-font-size-override`）。

仅当我们想先以零风险给方案查看器交付 PR 面板修复、之后再迁移到 A 时，才用 **A-lite**——但 A-lite 的影子样式表是技术债，所以除非时间受限，优先 A。

避免 B（prop 穿透的繁琐）和 C（重写）。

## 供规格的开放问题

1. **变量转换范围：** 全部约 9 个组件铺满 `--md-*` 层（干净），还是先处理显眼的（标题 + 块间距 + 正文），代码/表格/行内暂不动？
2. **变体交付：** 在 PR 容器上加 `.md-compact` 类，还是在新 `RenderedMarkdown` 上加 prop 来切换该类？（类更简单；prop 更易发现。）
3. **两档还是 N 档：** 只做 `default`（方案）+ `compact`（PR），还是从第一天就做成绑定 `configStore` 的连续刻度（v1 大概过度设计）？
4. **钉住默认值：** 确认方案查看器必须保持像素级一致（默认值 = 今天的字面值），所以这不能变成"顺手把方案也调一调"的改动。

## 涉及文件

- `packages/ui/components/BlockRenderer.tsx`、`InlineMarkdown.tsx`、`ListItemBody.tsx`
- `packages/ui/components/blocks/{CodeBlock,TableBlock,HtmlBlock,Callout,AlertBlock,proseBody}.tsx`
- `packages/ui/theme.css:107-110`（`--text-*` 阶梯）
- `packages/review-editor/index.css:282, 1275+`（`--diff-font-override` / `--diff-font-size-override` 先例）、`:869`（`.compact` 先例）
- `packages/ui/config`（`configStore` / `useConfigValue`）