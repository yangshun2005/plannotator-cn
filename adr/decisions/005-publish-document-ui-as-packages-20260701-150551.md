# 005. 将文档 UI 发布为 `@plannotator/ui` + `@plannotator/core`

日期：2026-07-01

## 状态

已接受

## 背景

商业版 Workspaces 应用是一个独立宿主应用，拥有自己的后端（自己的存储、身份、上传和实时能力），它需要 Plannotator 的文档 UI：Markdown 渲染引擎、标注/评论机制、Markdown 编辑器和主题。在实现这一点的同时，Plannotator 的开源体验不能改变。

当时有两种方案：

1. **复制/内置** —— 将 UI 源码 Fork 到宿主应用自己的仓库（此前曾尝试从零重新实现该 UI，结果破坏了应用，随后被回滚）。
2. **发布** —— 将 UI 做成可安装的包，并提供宿主覆盖接缝。

复制/内置方案在两个我们后来实测的事实上落败：

- **可复用部分深埋在包依赖图中。** `packages/ui` 大量从 `@plannotator/shared` 导入，而后者绑定 Node（fs/git/server）。复制方案必须手工切断这些导入，并在每次同步时重新切断。
- **主分支变化太快，不适合 Fork。** 仅打包 PR（#957）期间，该分支就**四次**变基到主分支（46 个提交，然后是 #979/#980、#981/#983、#878 + 0.21.4 发布）。内置副本会立刻落后四次同步；主分支上落地的每个 UI 功能（数学渲染、解析器加固、PR 上下文工作）都需要手工移植。

接缝优于手术，原因相同：一个覆盖点的默认值*就是*今天的代码，从而保持单一事实来源；而按消费者逐个修改调用点会制造两份。

## 决策

发布两个包，保留一个私有包，并通过接缝解耦——绝不重写：

- **`@plannotator/core`** —— 浏览器安全、零依赖的纯工具 + 类型，用 `git mv` 从 `@plannotator/shared` 中切出。CI 在不含 `@types/node` 的情况下对其进行类型检查，因此 `node:` 导入无法混入。已发布。
- **`@plannotator/ui`** —— React 组件、Hook、主题和 `configurePlannotatorUI()`。依赖 `core`。已发布。
- **`@plannotator/shared`** —— 保持私有；通过一行垫片（`export * from '@plannotator/core/x'`）重新导出移出的 core 模块，因此 Plannotator 约 99 个内部导入点均未改变。

**接缝模式：** UI 接触后端的每个位置都是带 setter 的模块级单例——`let impl = default; setX(); resetX(); getX()`——其默认值复现 Plannotator 的字面行为（通过 fetch 访问 `/api/*`）。Plannotator 不传任何东西，保持逐字节不变；宿主在启动时调用一次 `configurePlannotatorUI({...})`。已交付的接缝：存储、身份（含 `isEditable`）、图片解析、上传、文档预览、文件树、草稿、外部标注、AI 传输、服务器同步。

**用单例而非 React Provider——并附带明确的重新审视条件：** 模块级状态对客户端应用是安全的（每个浏览器一个用户）。在 SSR 下则是错误的，因为一个服务器进程为多个用户渲染。Plannotator 和 Workspaces 都不做 SSR。**如果将来为此 UI 添加 SSR，那就是引入 `<PlannotatorUIServices>` Provider 的触发条件**，并将 `configurePlannotatorUI` 变成其上的兼容垫片。在此之前不支付这份复杂度。

**法则**（护栏，尽可能通过社会约定和 CI 强制执行）：

1. 不要从零重新实现文档 UI——添加接缝。
2. 每个接缝的默认值必须复现今天的 Plannotator 行为。
3. `@plannotator/core` 保持浏览器安全和零依赖——不得有 `node:` 导入。
4. 在人类确认浏览器中行为一致之前，绝不删除可用的 Plannotator 代码。

**发布模型：** 两个包**与仓库版本同步**（当前 0.21.4）。它们发布 TypeScript 源码（由消费者打包）；`styles.css` 由 `prepack` 预编译。`workspace:*` 内部依赖在发布时必须解析为精确版本，因此流程是 `bun pm pack`，然后 `npm publish *.tgz --provenance --access public`，**先 `core`，后 `ui`**。首次发布在合并后从 `main` 手动执行；CI 发布任务是后续工作。

## 后果

- 宿主应用安装这些包，并通过 `configurePlannotatorUI()` 将自己的后端接入接缝（设置存储、带 `isEditable() => false` 的身份、上传、文档预览、标注传输），无需 Fork。消费者侧的决策记录在宿主应用自己的仓库中，而非此处。
- 发布的导出映射很宽；**可导入 ≠ 受支持**。受支持的导入允许列表和不受支持的“调用 Plannotator 本地服务器”列表位于 `packages/ui/HANDOFF.md`，与标注锚点 Schema 和编辑器/Yjs 状态放在一起。重构导出被有意推迟。
- v1 接受的已知粗糙边缘（记录在 HANDOFF.md 中）：`AITransport`/`FileTreeBackend` 泄漏 `Response` 对象；Markdown 编辑器尚不接受 CM6 扩展（实时协作将通过后续原子 PR 落地，在 `@plannotator/markdown-editor` 导入 monorepo 后，将 `extensions?` 贯穿两个包装层）。
- 未来每个“宿主需要不同行为”的请求都有一种被认可的形态：在 Plannotator PR 中添加一个可选接缝，其默认值为今天的行为。
- 版本同步意味着 UI 发布就是仓库发布；发布仍需所有者授权。

## 参考

- PR #957（`feat/pkg-document-ui`）
- `packages/ui/HANDOFF.md` —— 消费者交接（接缝目录、受支持导入、锚点 Schema）
- `packages/ui/README.md`、`packages/ui/CLAUDE.md` —— 代码旁的简版说明