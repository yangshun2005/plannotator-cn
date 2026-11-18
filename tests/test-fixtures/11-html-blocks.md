# 原始 HTML 块测试夹具

本夹具用于测试新的 CommonMark Type 6 HTML 块检测以及净化渲染器。每个块前后的标题和段落应正常渲染为 Markdown——HTML 块应渲染为真实 HTML。

## 1. GitHub 风格可折叠区块

<details>
<summary>前置条件</summary>

安装 **bun** 并克隆仓库。摘要行始终可见；正文仅在展开时渲染。在此展开块内选中文本应通过 web-highlighter 触发标注工具栏。

</details>

可折叠区块后的普通段落。应渲染为纯 Markdown，**粗体**和*斜体*格式保持不变。

## 2. 嵌套可折叠区块

<details>
<summary>外层区块</summary>

<details>
<summary>内层区块</summary>

嵌套 details 渲染为一个 HTML 块（CommonMark Type 6 在空行处终止，而非匹配的闭合标签）。

</details>

</details>

## 3. 含行内格式的原始引用块

<blockquote>
此引用块以原始 HTML 编写。应使用 theme.css 中的 <code>.html-block blockquote</code> 样式渲染——主色调左边框和斜体弱化前景色。
</blockquote>

## 4. section / header / footer 地标

<section>
<header><strong>区块标题</strong></header>

section/article/aside/header/footer 标签在允许列表中，应渲染为通用块容器。

<footer>区块页脚</footer>
</section>

## 5. 净化器冒烟测试

<details>
<summary>应剥离脚本和事件处理器</summary>

<script>alert('xss')</script>

<p onclick="alert('xss')">点击我——onclick 处理器应被 DOMPurify 剥离。</p>

<a href="javascript:alert('xss')">此 javascript: href 应被中和。</a>

<a href="https://plannotator.ai" rel="noopener" target="_blank">此安全的 https 链接应正常渲染。</a>

</details>

## 6. 非允许列表标签穿透

<xyz>此自定义标签不在块标签允许列表中，因此解析器应将其保留为段落文本，React 会转义尖括号。</xyz>

## 7. 空行分隔的相邻块

<details>
<summary>第一个</summary>
一。
</details>

<details>
<summary>第二个</summary>
二。
</details>

## 8. 以原始 HTML 编写的表格

<table>
  <thead>
    <tr><th>标签</th><th>是否允许</th></tr>
  </thead>
  <tbody>
    <tr><td>details</td><td>是</td></tr>
    <tr><td>script</td><td>否——已剥离</td></tr>
  </tbody>
</table>

## 结束

最后一段——确认 HTML 块不会吞掉后续内容。