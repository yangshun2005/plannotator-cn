# 规格说明：移动平台基础 — 第一阶段

日期：2026-08-12
状态：检查点 1A 和 1B 已实现；1B 已在实体 iPhone 上验收；下一步 1C
基线：从 `origin/main` 的 `ef49c701c23b867cec2a5d78343813ba89d2a025` 开始；在 1B 收尾前刷新至 `356b628b6fd0e956e058ec684b103e1e68311999`
实现分支：`codex-mobile-phase-1-foundation`，随后堆叠 `codex-mobile-phase-1b-input`

相关研究：[`adr/research/SPIKE-mobile-web-compatibility-20260812.md`](../research/SPIKE-mobile-web-compatibility-20260812.md)

## 阶段契约

第一阶段让 Plannotator 在任一评审体验针对移动端重新设计之前，先表现得像一个稳定的移动 Web 应用。

它为以下方面建立一份共享契约：

- 可见的 Safari 视口与软键盘；
- 刘海、灵动岛、圆角与 Home 指示条的安全区域；
- 可读的文本输入控件，不触发 iOS 聚焦缩放；
- 共享控件中的触摸目标尺寸与按压反馈；
- 代表性浮层中的聚焦、关闭与滚动行为；
- 使用精细指针和键盘时，桌面呈现保持不变。

访客模式为 **Operate**。评审者可能离开开发机、通过二维码进入，单手操作、网络不稳定，对浏览器界面故障缺乏耐心。期望的感受是平静而直接：产物保持稳定，输入跟随键盘，控件在评审者触摸处响应。

## 结果与证明

在第一阶段关卡，评审者必须能在 iPhone 或 iPad 上通过 Tailscale 打开 Plan 和 Code Review，旋转设备，显示和关闭软键盘，撰写并提交一条代表性评论，并在该评论流程中触达每个控件，同时应用画布不跳动、不缩放、不落后于系统 UI。

证明包括：

1. 视口环境计算与清理的自动化测试；
2. 两个单文件应用的生产构建；
3. 在约定参考宽度下的响应式浏览器证据；
4. 通过 Tailscale 在 Mobile Safari 上的实体设备验证；
5. 桌面回归验证，证明现有组合得以保留。

本阶段仅在实体设备评审通过后才算完成。仅靠模拟无法关闭本阶段。

## 范围边界

### 范围内

- Plan/文档与 Code Review 应用根节点，包括加载状态。
- 共享的 `@plannotator/ui` 环境与控件原语。
- 主 Plan 评论编辑器与主代码标注编辑器。
- 共享对话框关闭行为与代表性模态边界。
- Mobile Safari 与 iPadOS Safari；Chromium 响应式模式仍作为快速预检。
- 仅触摸的 iPad 与带指针/键盘的 iPad，不强制任一进入另一方的布局。

### 明确非目标

第一阶段**不**：

- 构建移动端文档/文件导航器（`MOB-001`）；
- 简化 Plan 头部、工具栏或评审操作（`MOB-015`–`MOB-019`）；
- 选择手机端 Code Review 组合、Unified 覆盖或 Guide 默认值（`MOB-002`、`MOB-003`、`MOB-020`、`MOB-021`）；
- 修改 Pierre 渲染、行选择或 `DiffViewer`（`MOB-006`）；
- 重新设计设置（`MOB-014`）；
- 解决原始 HTML 标注中任意作者页面的响应式问题（`MOB-022`）；
- 更改服务器 API、Tailscale 暴露、认证或会话生命周期；
- 优化单文件载荷（`MOB-023`）；
- 引入新的视觉语言、新设计系统或替换文档 UI。

这些留待后续阶段。若第一阶段补丁开始重排产品信息架构，则已越权。

## 设计与工程不变量

1. **内容破损决定布局；设备名称不决定。** CSS 处理几何。指针与悬停媒体特性处理输入可供性。JavaScript 仅在浏览器 API 暴露 CSS 无法可靠提供的状态时观察视觉视口。
2. **不禁止缩放。** 视口声明可添加 `viewport-fit=cover`，但不得设置 `maximum-scale=1` 或 `user-scalable=no`。
3. **不全局修改桌面偏好。** 第一阶段不基于设备宽度写入评审布局、差异呈现或侧边栏偏好。
4. **文档仍是权威。** 基础外壳可预留安全空间，但不得为掩盖产物溢出而重设样式或重排作者编写的 markdown、代码或原始 HTML。
5. **触摸尺寸可不同于视觉尺寸。** 16 px 图标可保持视觉紧凑，但只要存在 `any-pointer: coarse`，其不重叠的交互区域必须至少为 44×44 CSS px。这在混合 iPad 连接触控板后仍保留触摸安全。
6. **不可有重叠的不可见目标。** 当相邻控件无法在不重叠的情况下支持扩展伪命中区域时，实际布局必须腾出空间。精度不是可接受的移动端依赖。
7. **按压反馈立即开始。** 共享按钮在 pointer-down/touch-down 时暴露按下状态。悬停仅在 `(hover: hover) and (pointer: fine)` 下增强控件。
8. **安全区域归属唯一。** 应用根节点拥有顶部与侧边内边距。底边表面拥有自己的底部内边距。父子不得同时添加同一内边距。
9. **键盘适配不丢弃工作。** 打开、调整大小、旋转、后台化或关闭编辑器必须保留草稿，并将焦点/滚动返回到连贯位置。
10. **共享 UI 保持共享。** 两个应用都需要时，变更属于 `@plannotator/ui`。Plannotator 继续使用现有包，而非构建并行的移动组件集。

## 实现形态

### 检查点 1A — 视口与安全区域契约

#### 入口文档

更新生产单文件构建使用的源 HTML：

- `apps/hook/index.html`
- `apps/review/index.html`

视口声明变为：

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
```

生成的 Hook、Review、OpenCode 和 Pi 资产继承这些源构建。生成的 HTML 文件不手工编辑。VS Code webview 不是边到边的移动交付表面，不属于实体设备关卡。

#### 共享视口观察器

在 `packages/ui/hooks/` 下添加一个小型导出 Hook，暂定名 `useViewportEnvironment.ts`。它必须：

- 存在时优先使用 `window.visualViewport`；
- 回退到 `window.innerWidth` 和 `window.innerHeight`；
- 在 `document.documentElement` 上写入 CSS 自定义属性，而非在每次视觉视口事件时触发 React 渲染；
- 用一个 `requestAnimationFrame` 合并 `resize` 和 `scroll` 更新；
- 清理监听器、待处理的动画帧及其拥有的属性；
- 幂等或引用计数，使两个挂载的消费者不会移除彼此的文档级环境；
- 容忍测试、类 SSR 导入及无 `visualViewport` 的浏览器；
- 暴露纯计算函数，使几何无需 Safari 即可单元测试。

观察器拥有的契约：

```css
--pn-viewport-width
--pn-viewport-height
--pn-viewport-offset-top
--pn-viewport-offset-left
--pn-keyboard-inset
```

主题拥有的安全区域契约直接由 CSS `env()` 值设置：

```css
--pn-safe-top
--pn-safe-right
--pn-safe-bottom
--pn-safe-left
```

`--pn-keyboard-inset` 是诊断/布局证据，不承诺每个虚拟键盘都是矩形。它由布局视口减去可见视口计算，下限为零。应用画布使用可见高度；不因偏移变化平移整个文档。

#### 共享 CSS

在 `packages/ui/theme.css` 中添加刻意命名的移动环境工具类：

- `.pn-app-viewport` 使用观察到的可见高度，在观察器值可用前依次回退 `100vh`、`100dvh`；
- `.pn-safe-top`、`.pn-safe-inline` 和 `.pn-safe-bottom` 仅消费各自拥有的内边距；
- `.pn-viewport-bounded` 将浮层高度限制在可见视口内；
- 所有值在无安全区域内边距的设备上保持零成本行为。

避免第二份移动样式表，避免同一变量的应用特定副本。

#### 应用集成

在以下根节点调用一次该 Hook：

- `packages/editor/App.tsx`
- `packages/review-editor/App.tsx`

将加载与应用根节点上的 `h-screen` 替换为 `.pn-app-viewport`。本检查点不替换所有历史 `100vh` 出现。仅授权主评论旅程中的根节点与浮层；无关弹出层按所属阶段盘点。

#### 检查点评审

在进入输入工作前，在实体设备上评审：

- 展开/收起 Safari 浏览器界面；
- 旋转竖屏 ↔ 横屏；
- 从全新 Tailscale URL 打开并从后台返回；
- 检查刘海/圆角间隙；
- 对比 plan 与 review 根高度与可见视口。

此处不评判产品布局意见。我们检查舞台本身是否可信。

### 检查点 1B — 键盘与文本输入契约

#### 移动可编辑样式

为用户编写的文本控件定义一个显式共享标记，暂定 `data-pn-mobile-editable`。当存在 `any-pointer: coarse` 或视口紧凑时，它保证计算字体大小至少 16 px。不得放大代码标签、计数器或只读差异文本。

在第一阶段主旅程中采用该标记：

- `packages/ui/components/CommentPopover.tsx` 中的 plan/文档评论编辑及其直接使用的共享 textarea 原语；
- plan 标注面板中属于同一提交旅程的直接评论输入；
- `packages/review-editor/components/AnnotationToolbar.tsx` 中的代码标注编辑；
- 从该旅程进入时 `packages/review-editor/components/ExpandedCommentDialog.tsx` 中的展开代码评论编辑。

搜索、设置、Ask AI、代理提示、源编辑和建议编辑仍盘点，但不静默拉入本阶段。其所属表面阶段必须采用同一标记，而非发明不同修复。

#### 放置边界

提取共享的纯可见视口边界辅助函数，而非散布新的 `visualViewport` 读取。仅在主编辑器放置路径中替换 `window.innerHeight` / `window.innerWidth`：

- `packages/ui/components/CommentPopover.tsx`；
- `packages/review-editor/hooks/useAnnotationToolbar.ts`；
- `packages/review-editor/components/AnnotationToolbar.tsx`。

编辑器必须保持在可见视口内并留有安全内边距。当键盘为锚定弹出层留下空间不足时，应使用现有模态/浮层呈现并限制在可见视口内；发明最终移动底部抽屉设计属于 Plan 或 Code Review 阶段。

#### 聚焦与关闭

- 除非打开本身是显式“写入”操作，否则不要在粗指针上自动聚焦新打开的文本字段。
- 浮层关闭时，若触发器仍挂载，将焦点恢复到其触发器。
- 硬件键盘下 Escape 仍可用。
- 背景关闭不得提交或擦除非空草稿。
- 提交保持显式，并在可能重复提交的进行中请求期间禁用。

#### 检查点评审

在 iPhone 和 iPad 上：

- 打开 plan 评论，输入多行，旋转，关闭并重新打开键盘，然后提交；
- 打开代码标注评论，输入多行，旋转，关闭并提交；
- 验证无 Safari 聚焦缩放；
- 验证光标、草稿、关闭控件和提交控件仍可触达；
- 验证关闭键盘不会使画布停留在错误缩放或滚动位置。

### 检查点 1C — 共享触摸与对话框契约

#### 触摸目标

在 `packages/ui/theme.css` 中添加共享触摸目标标记与令牌：

```css
--pn-touch-target: 44px;
[data-pn-touch-target]
```

当存在 `any-pointer: coarse` 时，标记强制 44 px 最小块尺寸，对于仅图标控件强制 44 px 最小行内尺寸。在仅有精细指针的设备上无几何效果。

默认在两个现有共享按钮原语中采用：

- `packages/ui/components/ui/button.tsx`（规范 Base UI 按钮）；
- `packages/ui/components/core/button.tsx`（仍在使用的旧共享原语）。

同时应用于 `packages/ui/components/ui/dialog.tsx` 中的共享对话框关闭控件。手写的 Plan 与 Code Review 工具栏按钮在第一阶段不批量转换；每个后续表面阶段必须在扩展每个目标前解决其控件分组与噪声。

若默认采用暴露出无法容纳正确目标的布局，将其记录为表面阶段的证据。不要仅为保留拥挤移动行而添加未记录的紧凑逃生口。

#### 按压、悬停与动效

- 共享按钮获得即时克制的按压反馈，不改变字重或布局。
- 在触及的原语中，仅悬停的视觉变化尽可能限制在精细悬停能力指针下。
- 任何新过渡命名确切属性，并有减少动效等效项。
- 本基础阶段不添加弹簧、滑动关闭或半透明抽屉系统。这些需要交互原型及其所属表面决策。

#### 对话框边界

更新共享对话框原语，使其宽度和最大高度考虑安全区域与可见视口。关闭可供性必须在 320 px 宽度和键盘可见时仍可触达。现有 Base UI 焦点包含与恢复仍是权威。

本阶段不将所有自定义历史模态转换为共享对话框。记录异常以供后续加固。

#### 检查点评审

- 在粗指针模拟和实体设备上测量代表性共享按钮与对话框关闭。
- 确认即使命中区域增大，视觉字形仍适当紧凑。
- 确认相邻目标不重叠。
- 确认触摸按压反馈无粘滞悬停状态。
- 确认桌面键盘与指针导航不变。

## 预期文件映射

实现应接近此映射：

| 区域 | 预期文件 | 职责 |
|---|---|---|
| 入口视口 | `apps/hook/index.html`、`apps/review/index.html` | 边到边选择加入而不禁用缩放 |
| 共享环境 | `packages/ui/hooks/useViewportEnvironment.ts`、聚焦测试文件 | 视觉视口观察与纯几何 |
| 共享样式 | `packages/ui/theme.css` | 视口、安全区域、可编辑与触摸目标契约 |
| 共享控件 | `packages/ui/components/ui/button.tsx`、`packages/ui/components/core/button.tsx`、`packages/ui/components/ui/dialog.tsx` | 粗指针目标、按压状态、有界对话框 |
| Plan 集成 | `packages/editor/App.tsx`、`packages/ui/components/CommentPopover.tsx`，可能直接共享的标注输入 | 根节点与主 plan 评论旅程 |
| Review 集成 | `packages/review-editor/App.tsx`、`packages/review-editor/hooks/useAnnotationToolbar.ts`、`packages/review-editor/components/AnnotationToolbar.tsx`、`ExpandedCommentDialog.tsx` | 根节点与主代码评论旅程 |

此映射外的文件需在实现交接中提供简短范围理由。预期无服务器文件或 Pierre/DiffViewer 文件变更。

## 状态与边界情况矩阵

| 状态 | 第一阶段要求行为 |
|---|---|
| 初始手机加载 | 根节点填满可见视口，无水平页面溢出 |
| Safari 界面展开/收起 | 根节点跟随可见高度，无持续空白或裁剪条 |
| 键盘打开 | 活动编辑器保持有界；输入保持正常页面缩放 |
| 键盘关闭 | 先前产物滚动与页面缩放保持连贯 |
| 编辑器打开时旋转 | 草稿保留；浮层重新计算可见边界 |
| 设备锁定/后台 | 草稿保留；返回不留下过期视口几何 |
| iPad 硬件键盘 | 焦点与 Escape 行为仍可用；不强制仅触摸组合 |
| iPad 连接触控板 | 精细指针悬停可增强控件；触摸仍有效 |
| 安全内边距为零 | 桌面与矩形移动视口无幻影内边距 |
| 无 Visual Viewport API | `innerWidth` / `innerHeight` 回退产生可用应用 |
| 减少动效 | 新按压/浮层反馈不引入动效依赖 |
| 200% 布局压力 | 主编辑器控件仍可触达；文本不被裁剪 |

## 验证计划

### 自动化

至少运行：

```bash
bun run typecheck
bun test <new viewport environment tests>
bun test packages/ui/components/CommentPopover*.test.tsx
bun run build:review
bun run build:hook
```

为任何新按钮/对话框 prop 或属性行为添加聚焦组件测试。不要仅用宽泛快照作为视口几何的唯一证明。

### 响应式浏览器预检

在以下尺寸运行两个应用：

- 320×568；
- 390×844；
- 430×932；
- 768×1024；
- 820×1180；
- 1180×820；
- 1440×900 桌面对照。

在每个相关宽度检查浅色与深色主题、根溢出、计算输入字体大小、目标几何、模态边界、控制台错误与草稿保留。

### 通过 Tailscale 的实体设备关卡

阶段关卡所需设备：

- 一台当前刘海 iPhone，竖屏与横屏；
- 一台 11 英寸级 iPad，竖屏与横屏；
- iPad 先仅触摸，然后如有触控板/键盘。

所需网络/会话条件：

- 从公布的 HTTPS URL 或二维码打开；
- Wi-Fi tailnet 路径；
- 设备后台与恢复；
- 返回时宿主会话仍存活；
- 无 Funnel 或公开暴露。

本阶段宿主命令保持显式：

```bash
plannotator annotate <fixture.md> --tailscale --gate
plannotator review --tailscale
```

自动 plan Hook 在本阶段不获得 Tailscale 行为。

### 桌面回归

在 1440×900 精细指针下：

- 将 Plan 与 Review 外壳截图与基线对比；
- 验证键盘快捷键、焦点环、弹出层与对话框；
- 验证所有内边距为零时无安全区域间隙；
- 验证持久化面板与差异