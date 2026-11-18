# 页内锚点导航测试夹具

本夹具覆盖 slugger 生成的全部标题形态——ASCII、Unicode、连续标点、数字——并测试 `[text](#slug)` 链接，以及嵌入各类块级元素（段落、列表项、引用块、警告块、指令、表格、原始 HTML 块）中、会触发 `onNavigateAnchor` 回调的锚点。

点击下方任一链接，应在粘性滚动视口内平滑滚动到目标标题。均不应触发"共享计划无法加载"对话框。

---

## 目录

- [纯 ASCII 标题](#plain-ascii-heading)
- [含冒号、逗号和标点的标题！](#heading-with-colons-commas-and-punctuation)
- [标题 — 含长破折号](#heading-with-em-dash)
- [Café ☕ résumé](#café-résumé)
- [中文 标题](#中文-标题)
- [Español: año nuevo](#español-año-nuevo)
- [Русский заголовок](#русский-заголовок)
- [123 纯数字开头](#123-numbers-only-start)
- [大写标题转为小写](#uppercase-heading-stays-lower)
- [含 `inline code` 和 **bold** 的标题](#heading-with-inline-code-and-bold)
- [Very-Very-Very-Long-Heading-With-Many-Hyphens-To-Stress-The-Slugger](#very-very-very-long-heading-with-many-hyphens-to-stress-the-slugger)
- [原始 HTML 标题：`<h2 id="MySection">`](#MySection)
- [重复](#duplicate)
- [重复](#duplicate-1)

---

## 不同上下文中的锚点

段落链接：跳转到 [Café ☕ résumé](#café-résumé)。

- 列表链接：跳转到 [中文 标题](#中文-标题)
- 嵌套列表链接：
  - 深层链接到 [纯数字开头](#123-numbers-only-start)

> 引用块链接：前往 [Español: año nuevo](#español-año-nuevo)。

> [!NOTE]
> 警告块链接：前往 [Русский заголовок](#русский-заголовок)。

> [!WARNING]
> 结合 **bold** 和 *italic* —— 链接到 [大写标题转为小写](#uppercase-heading-stays-lower)。

:::tip 指令块
指令容器链接到 [标题 — 含长破折号](#heading-with-em-dash)。
:::

| 目标 | 链接 |
|---|---|
| ASCII | [前往](#plain-ascii-heading) |
| 标点 | [前往](#heading-with-colons-commas-and-punctuation) |
| Unicode 重音 | [前往](#café-résumé) |
| CJK | [前往](#中文-标题) |
| 西里尔字母 | [前往](#русский-заголовок) |
| 数字开头 | [前往](#123-numbers-only-start) |
| 原始 HTML id | [前往](#MySection) |
| 重复第二项 | [前往](#duplicate-1) |

<details>
<summary>含锚点的原始 HTML 块（点击展开）</summary>

原始 HTML 中的原生 `<a href="#...">` 链接也应经过滚动处理器，而不是触发浏览器原生跳转、错过滚动视口。

<ul>
  <li><a href="#plain-ascii-heading">HTML 列表 → ASCII 标题</a></li>
  <li><a href="#café-résumé">HTML 列表 → Unicode 标题</a></li>
  <li><a href="#MySection">HTML 列表 → 原始 HTML id</a></li>
</ul>

</details>

---

## 纯 ASCII 标题

应从 `#plain-ascii-heading` 解析。

## 含冒号、逗号和标点的标题！

连续的非字母数字字符折叠为单个 `-`。

## 标题 — 含长破折号

长破折号不应产生双连字符。

## Café ☕ résumé

Unicode 字母（重音）由 `slugifyHeading` 保留。咖啡 emoji 变为分隔符。预期 id：`café-résumé`。

## 中文 标题

CJK 字符视为 Unicode 字母，在 slug 化后保留。预期 id：`中文-标题`。

## Español: año nuevo

ASCII、冒号和 `ñ` 的混合。预期 id：`español-año-nuevo`。

## Русский заголовок

西里尔字母保留。预期 id：`русский-заголовок`。

## 123 纯数字开头

标题以数字开头——slugger 不应剥离前导数字。预期 id：`123-numbers-only-start`。

## 大写标题转为小写

标题文本为大写；slugger 转为小写。预期 id：`uppercase-heading-stays-lower`。

## 含 `inline code` 和 **bold** 的标题

行内 Markdown 在 slug 化前被剥离。预期 id：`heading-with-inline-code-and-bold`。

## Very-Very-Very-Long-Heading-With-Many-Hyphens-To-Stress-The-Slugger

连续已有的连字符应干净折叠。预期 id：`very-very-very-long-heading-with-many-hyphens-to-stress-the-slugger`。

## <a name="MySection"></a>原始 HTML 标题：`<h2 id="MySection">`

严格来说不是 Markdown 标题 id——本节使用行内 HTML 锚点，使 `#MySection` 通过原生 `name="MySection"` 属性解析。测试大写 / 非 slug 化的 id 路径。

## 重复

首次出现——裸 slug `duplicate`。

## 重复

第二次出现——由 `buildHeadingSlugMap` 去重为 `duplicate-1`。

---

## 回归检查

- 在 URL 中带 `#café-résumé` 加载本夹具，应自动滚动，且不触发共享计划错误对话框。
- 更改 hash（例如点击上方链接）应触发 `hashchange` 并滚动，同样不出现错误对话框。
- *确实*看起来像共享载荷的 hash（长、混合大小写 base64url）仍应路由到共享加载器——本夹具不包含任何此类 hash，因此测试本文件时共享路径不应触发。