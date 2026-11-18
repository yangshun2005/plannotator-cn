# 移动端触摸范围选择技术探索

**状态：** Code Review 交互已批准实施；Plan 交互探索仍未关闭

**分支：** `codex/mobile-touch-selection`

**基线：** `origin/main` 位于 `eb6a59e2dc6e418c1c4ed7830ee210091d754061`
**涉及界面：** Markdown Plan Review 与基于 Pierre 的 Code Review

## 背景

已发布的移动端基础能力让单个 Plan 块或 diff 行可以方便地标注，但扩展该目标仍依赖桌面式手势：

- 在 Drag 模式下，Safari 文本选择是跨多个 Plan 块的唯一方式。它会弹出系统原生的 Copy / Find Selection UI，并可能阻止 Plannotator 自身的操作接收下一次点击。
- Pierre 已支持通过 Pointer Events 直接拖动行号。在 DiffsHub 中的实际使用确认该手势在移动端可用；Plannotator 剩余的差异在于松手后立即打开编辑器，而不是保留范围并暴露 Pierre 的上下文行号栏操作。

目标是提供一种触摸原生的方式来选择一个连续范围。它不是新的标注模型，也不是通用移动端文本编辑器。

## 改变方向的产品反馈

首个原型的 **Extend selection** 和 **Adjust lines** 命令被否决。它们在初始目标和实际范围手势之间增加了一次模式切换，使交互比被替代的方案更慢、更不直接。这不是文案或呈现问题；命令中介模型本身就是错的。

后续实验必须满足更严格的约定：

- 扩展范围前不需要预备命令；
- 手指移动过程中持续提供视觉反馈；
- 松手即提交目标并暴露常规标注操作；
- 在活动选择控件之外仍可正常滚动；
- Plan 与 Code Review 的内容模型不同，可以使用不同的获取手势。

## 平台实际提供的能力

### Safari 原生文本选择

Safari 掌管长按文本选择、可拖动的前后手柄、放大镜以及 Copy / Find 菜单。Web Selection API 让 Plannotator 可以通过 `selectionchange` 观察并保留结果范围，但不暴露选择手柄 UI，也没有受支持的扩展点可向 Safari 菜单添加 Plannotator 操作。

`-webkit-touch-callout: none` 是 Safari 用于长按菜单的非标准控制项。它不能证明系统选择手柄仍可用，且 WebKit 的长按/放大镜行为因版本而异。因此，原生手柄、抑制菜单和自定义 Plannotator 操作的任何组合都需要在真机 Safari 上实验，不能仅靠代码得出结论。

主要依据：

- [W3C Selection API](https://www.w3.org/TR/selection-api/) 定义了文档选择、`selectstart` 和 `selectionchange`；它不暴露系统手柄或菜单定制。
- [W3C Pointer Events](https://www.w3.org/TR/pointerevents/) 指出 `touch-action` 控制浏览器平移/缩放，不控制文本选择或高亮。
- [Apple Safari CSS 参考](https://developer.apple.com/library/archive/documentation/AppleApplications/Reference/SafariCSSRef/Articles/StandardCSSProperties.html) 将 `-webkit-touch-callout` 记录为 Safari 专有的菜单开关。
- [WebKit bug 231161](https://bugs.webkit.org/show_bug.cgi?id=231161) 区分了文本选择（`user-select`）与长按菜单/放大镜行为，说明为何需要按版本限定设备测试。

### Pierre 行范围选择

`@pierre/diffs` 1.3.2 已提供所需的直接手势。在行号列按下指针即播种范围，文档级 Pointer Events 跨行跟踪，绘制中的选择持续更新，指针抬起时发出 `onLineSelectionEnd`。行号栏已是 `user-select: none` 和 `touch-action: none`，因此该路径既不触发原生文本选择，也不会把活动拖动交给页面滚动。

在 `origin/main` 基线上，Plannotator 已在 File 和 All Files 视图中设置 `enableLineSelection: true`，并将 `onLineSelectionEnd` 接入现有标注工具栏。下面的实现只改变紧凑触摸下的手势后路由。范围状态和交互引擎仍归 Pierre 所有。

## 方法

1. 在引入状态前，追踪现有的获取、预览、草稿、提交、恢复和导出路径。
2. 复用各标注流水线已接受的规范范围。
3. 将选择与文本输入分离：选择或调整目标不得聚焦 textarea 或唤起软键盘。
4. 将新组合逻辑置于共享的紧凑触摸判定之后：`(max-width: 1024px) and (pointer: coarse)`。
5. 将桌面精细指针行为视为对照。现有点击、拖动、键盘、工具栏和编辑器行为必须保持不变。
6. 以渲染浏览器检查作为预检，以真机 iPhone/iPad Safari 作为发布权威。

## 现有约定

### Plan Review

`usePinpoint` 通过与 Vim 导航相同的有序 `SemanticTargetGraph` 解析点击。`useAnnotationHighlighter` 已接受可跨块边界的 DOM `Range`，web-highlighter 将单个来源序列化为带起止元数据的形式。因此，注释、红线、草稿、导出、重载恢复和侧边栏导航都不需要为连续块范围引入第二套标注 Schema。

缺失的接缝是获取。目前 Pinpoint 在首次选择打开工具栏或编辑器后即被禁用。

### Code Review

`SelectedLineRange` 已表示多行目标。`useAnnotationToolbar` 提取选中代码并提交相同的起止范围，而 `DiffViewer` 和 `AllFilesCodeView` 都将 `pendingSelection` 作为受控选择投影回 Pierre。

Pierre 当前的交互管理器已使用 Pointer Events 并支持行号栏拖动。被否决的原型错误地将非拖动替代方案视为缺失接缝。已批准的 Code Review 实现保留 Pierre 的直接手势，不改变范围类型、命中几何或渲染器内部。

## 被否决的共享交互模型

首个原型使用显式两步调整：

1. 用户当前的单目标成为固定锚点。
2. 用户调用本地 **Extend** / **Adjust lines** 操作。
3. 活动编辑器/编辑器让出，但不丢弃草稿。一个考虑安全区的小型说明界面提示点击位置并提供 Cancel。
4. 同一文档或文件中的下一个合格目标成为另一端点。
5. Plannotator 预览规范化后的连续范围，并返回之前的工具栏/编辑器。

这有意采用端点选择，而非点击切换任意项。其数据模型合理，但交互不够符合人体工学，无法继续。该实现仍可作为证据，说明两条标注流水线都已接受连续范围；它不是提议的 UI。

## 直接操作实验

### Code Review — DiffsHub 参考与已批准行为

实际反馈确认多行选择在独立 DiffsHub 应用（Pierre 仓库的本地检出）中运行良好。源码对比显示，DiffsHub 和 Plannotator 已启用相同的 Pierre 约定：`enableLineSelection`、`enableGutterUtility`、`onLineSelectionEnd` 和 `onGutterUtilityClick`。

关键差异在选择之后：

- DiffsHub 的 `onLineSelectionEnd` 只保留受控的 `selectedLines` 范围并更新其行链接。它不打开编辑器。
- DiffsHub 的 `onGutterUtilityClick` 单独为选中范围创建草稿注释。
- Plannotator 目前将两个回调都路由到 `ToolbarHost.handleLineSelectionEnd`；因此紧凑触摸在范围手势结束时立即打开展开的编辑器。

紧凑触摸行为现已与 DiffsHub 的选择优先过渡一致：

1. 触摸行号并在同一动作中开始拖动。
2. 手指跨行时，选中范围 1:1 绘制在手指下方。
3. 松手后多行范围仍保持选中，不打开键盘或编辑器。
4. 激活上下文行号栏注释工具，为该范围打开常规反馈编辑器。

没有预备的 **Adjust lines** 命令，也没有常驻说明。行号栏工具是直接选择后的上下文操作，不是选择前的模式切换。由于现有 DiffsHub 几何已通过真机多行选择检查，Plannotator 应首先采用状态过渡对齐，不改变 Pierre 的 Shadow DOM、手势识别器或行号栏尺寸。几何变更需要单独的真机证据。

实现决策：紧凑触摸下 `onLineSelectionEnd` 保留受控范围和活动文件，不挂载编辑器。现有 `onGutterUtilityClick` 路径仍是显式书写操作，并打开编辑器。精细指针桌面保留现有的选择到编辑器过渡。这不改变任何 Pierre 选项、选择器、CSS 变量、手势或范围类型。

验证必须覆盖行号栏附近的垂直页面滚动意图、代码水平滚动、分栏和统一视图、正向和反向拖动、边缘自动滚动、单行选择、选择替换、不书写直接关闭，以及编辑器打开期间选择保持。

### Plan — 候选 A：原生范围，操作移开

1. 长按文本，使用 Safari 自带手柄选择所需词语或块。
2. 从 `selectionchange` 保留实时范围。
3. 在考虑安全区的底部 dock 中呈现 Plannotator 标注操作，与 Safari 选择菜单空间分离。
4. 在操作 pointer-down 时捕获已保存范围，避免 Safari 折叠可见选择导致目标丢失。

如果原生手柄能跨渲染 Markdown 扩展，且底部操作可点击而不禁用普通复制、查询、辅助功能或滚动，则该候选胜出。抑制 Safari 菜单是可选的实验单元，不是默认假设。

### Plan — 候选 B：语义范围手柄

1. 普通 Pinpoint 点击像今天一样选择一个语义块。
2. 选中范围暴露直接的后置手柄；没有 **Extend** 命令。
3. 将该手柄拖过另一语义块，连续扩展或收缩连续范围。
4. 手柄保持附着手指时，靠近边缘自动滚动文档。
5. 松手后在可见端点返回常规标注工具栏。

只有手柄拥有 `touch-action: none`；文档其他位置保留原生垂直滚动。这是自定义 Web UI，但借用了熟悉的前后手柄模型，并将手指直接映射到选中范围。

候选 B 优于全文档拖动识别器，因为在 Plan 中接管任意位置的垂直拖动会与其主要阅读/滚动手势冲突。按住再拖动的识别器优先级也较低，因为它与 Safari 长按选择竞争，并引入消歧延迟。

## 历史被否决的 Plan 原型：块端点选择

适用条件：

- 紧凑触摸布局；
- Pinpoint 输入方式；
- 同一渲染 Markdown 文档中的普通含文本语义目标；
- 存在待处理选择工具栏。

流程：

1. 在 Pinpoint 中点击段落或列表项。
2. 点击选择工具栏中的 **Extend**。
3. 工具栏让出；原高亮保留为锚点。
4. 正常滚动并点击最后一个段落或列表项。
5. 无论点击方向如何，从较早块的第一个边界到较晚块的最后一个边界构建 DOM 范围。
6. 常规工具栏返回并覆盖合并后的高亮。注释、删除、快速标签、复制、取消、草稿、提交和重载路径保持现有行为。

首原型限制：

- 仅一个连续范围；
- 仅文本块可作为端点；代码、数学、表格和原始 HTML 保留其专用选择路径；
- 调整从选择工具栏开始，在注释编辑器打开之前；
- 切换文档或离开 Pinpoint 会取消调整并恢复原始待处理目标。

## 历史被否决的 Code Review 原型：行端点选择

适用条件：

- 紧凑触摸布局；
- 新建行注释草稿，而非现有标注编辑或仅 token 标注；
- 活动文件和普通行范围。

流程：

1. 像今天一样选择一行并开始注释。
2. 点击范围标题旁的 **Adjust lines**。
3. 编辑器让出，不保存、不清空、不聚焦任何内容。紧凑说明界面显示 **Tap the last line** 并提供 Cancel。
4. 点击同一文件中的一行。Plannotator 将该端点与原始锚点合并，并通过受控 `selectedLines` 更新 Pierre。
5. 同一草稿编辑器返回，其文本、标签、装饰、建议状态和光标数据保持不变。标题反映新范围。

首原型限制：

- 仅同一文件；
- 仅同一 diff 侧，匹配标注/导出模型的单侧原始代码提取；
- 编辑现有已提交标注时不可调整范围；
- 不尝试替换 Pierre 现有的鼠标/触控板拖动或 Shift 扩展。

## 历史被否决原型的状态与取消规则

- 在点击有效端点前，已提交选择不变。
- Cancel 返回之前的工具栏/编辑器，保留原始范围和草稿。
- 存在硬件键盘时，Escape 执行相同取消。
- 设备旋转和视觉视口变化保留调整状态。
- 切换活动 Plan 文档、审查文件、diff 族或编辑会话会取消调整。
- 点击不合格端点不会丢弃草稿，也不会创建部分范围。
- 开始调整绝不打开软键盘。返回编辑器时，粗指针不会自动聚焦。

## 历史被否决原型的视觉与无障碍约定

- 锚点和候选范围使用现有选择颜色；不引入新的常驻文档装饰。
- 临时说明界面位于 Safari home indicator 内边距之上，且绝不接管页面滚动。
- 紧凑触摸布局中操作至少为 44 x 44 CSS 像素。
- 状态文本通过 polite live region 播报。
- 颜色不是唯一状态指示：说明文本和范围标签标识调整模式。
- 减少动态效果模式移除任何让出/重入过渡。

## 桌面非回归约定

在精细主指针下，新 Props 和状态不生效：

- Plan Pinpoint 和 Drag 选择行为与 `origin/main` 完全一致。
- Pierre 行号拖动、Shift 扩展、鼠标选择、悬停工具、分栏/统一渲染和键盘快捷键不变。
- 工具栏和编辑器几何不变。
- 没有媒体查询使用 `any-pointer: coarse`。

## 验证矩阵

自动化覆盖：

- 正向和反向文档顺序的范围规范化；
- Plan 取消、无效目标、模式/文档变更和来源替换；
- Code Review 草稿保留、同侧范围合并、无效侧/文件、取消和受控选择投影；
- 手机/iPad 配置上的紧凑触摸判定，以及窄精细指针和混合主鼠标配置上的不生效行为；
- 调整期间无 focus 调用。

渲染预检：

- 320 x 568 和 390 x 844 紧凑手机配置；
- 768 x 1024 iPad 竖屏和 1024 x 768 iPad 横屏；
- 1280 x 720 和 1440 x 900 桌面对照；
- 长 Plan、长 diff、从首屏下方开始的范围，以及调整期间旋转。

真机门槛：

- iPhone Safari：调整期间滚动时浏览器 chrome 折叠；不出现原生选择菜单或放大镜；键盘保持关闭，直到用户点击 textarea。
- iPad Safari：手指和触控板路径均可用；触控板作为主指针时不继承紧凑触摸组合。
- Code Review：编辑器返回后选择仍保持绘制，重载后提交的标注覆盖预期行。

## 下一步决策

Code Review 已有批准的直接手势，不再需要端点选择实验。真机 Safari 验证仍是其紧凑触摸过渡的发布门槛。Plan Review 仍需单独的直接操作决策；被否决的 **Extend selection** 命令不得以不同文案回归。

## 实现证据

- 共享路由策略区分 Pierre 的范围完成手势与其上下文行号栏注释操作。
- File 和 All Files 视图使用相同的紧凑触摸规则：手势完成保留选中范围；行号栏操作打开现有编辑器。
- All Files 路由在发布范围前还提交所属项和活动文件，因此相同编号的范围不会继续绘制在先前活动文件上。
- 精细指针桌面保留现有的选择到编辑器过渡。
- 聚焦策略和渲染 All Files 生命周期测试通过，包括桌面对照。根类型检查和两个生产单文件构建（`apps/review` 和 `apps/hook`）通过。
- 未改变任何 Pierre 选项、不安全 CSS、Shadow DOM 选择器、手势识别器、包版本、工具栏几何或编辑器几何。真机 iPhone/iPad Safari 仍是最终手势检查的权威。