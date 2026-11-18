# 007 — 便携式引导式评审

日期：2026-08-15
状态：已接受（设计），实现规范见 `adr/implementation/portable-guided-reviews.md`
取代：分支 `guided-review-html-export`（`2cf18cc3`，从未合并）——部分内容被保留，主干已重新设计。

## 背景

引导式评审（Guided Review）是 Plannotator 产出的最易分享的产物：它按章节有序地走查一个变更集，并内联展示真实 diff。目前它只存在于运行中的 Plannotator 会话里。我们希望把它作为单个文件交给别人，外观与在应用中完全一致；并且——日后——把它托管在某个 URL 上，让未来的平台基于同一格式构建。

早先的尝试把*整个评审应用*导出进 HTML（17–18 MB），或引用营销 CDN 上一个可变的 `v1/` 查看器，且只能导出当前内存中的引导。这两点都与目标冲突。哪些内容被保留，见 `adr/implementation/portable-guided-reviews.md` 的“Salvage”一节。

## 决策

以下每条编号决策都对应实现规范中的一项验证检查（§Verification matrix）。

### D1 — 产物是一个小 HTML 文件，其大小取决于 diff，而非应用
导出文件包含引导 JSON、精确的 diff、来源信息和纯文本回退正文。它引用 `guides.show` 上的渲染器，而不内嵌渲染器。**不限制 diff 大小**——巨大的 diff 会产生巨大的文件，这是用户的决定。我们拒绝的是打包应用造成的数兆字节*基线*。

### D2 — 一个渲染器，两个宿主：应用内引导与便携式引导是同一份代码
引导渲染链（`GuideView → GuideSectionCard → GuideFileCard → GuideViewportManager → AllFilesCodeView`，基于 `@pierre/diffs`，外加两个无依赖的 markdown 渲染器）移入包 `@plannotator/guide-viewer`，Plannotator 的评审应用从那里导入。“无视觉漂移”因此是结构性的，而非承诺。`AllFilesCodeView` 在 `readOnly` 模式下复用，而非替换为精简的同类组件；标注/编辑机制在只读宿主中代码分割出去。已否决：单独的轻量 diff 渲染器（会在第一次 Pierre 升级时漂移，并重新推导 #1158 视口契约）。

### D3 — 现在只读，日后反馈，格式为其预留
便携式引导是只读的（滚动、展开/折叠章节、跳转到文件、浅色/深色、会话内“已评审”切换但不持久化）。评审者标注和 PR 评论线程在 v1 中**不**导出。快照格式预留了 `annotations` 块，查看器保留其标注槽位，以便日后添加共享反馈时不破坏格式或渲染器。

### D4 — 仅引导界面
便携式文件渲染引导界面——绝不渲染完整评审外壳（文件树、全部文件视图、搜索、批准/反馈）。文件中任何内容都不得尝试发起服务器调用。

### D5 — 快照是带版本、严格、自描述的文档
载荷（v1）：`guide`（title、intent、sections[{title, overview, diffs[{file, summary}]}]、unplacedFiles、reviewed[]）、`review`（rawPatch、gitRef 标签、diffType、base）、`source`（kind local|pr|workspace|commit、repo、branch、headSha、pr{url,number,title,platform}?、commitSha?）、`generator`（engine、model、generatedAt、用于溯源的 **customInstructions**）、`theme?`（导出器调色板提示）。解析时拒绝未知字段；格式每个版本有一个 fixture 和一个兼容性测试。所有引导来源（本地 diff 类型，含提交/工作树；PR 层/全栈；多仓库工作区）都可导出，且头部必须清楚说明引导是*关于*什么的。

### D6 — 在引导任务启动时捕获 diff，而非用户导出时
引导描述的是它生成时所针对的 diff；导出时屏幕上的 diff 可能已变化。Plannotator 在任务上捕获启动时的补丁（与 `changedFilesSnapshot`/`guideContext` 已使用的同一通道），并存储在已保存引导旁（`{id}.patch`）。一次一个引导；不是“历史引导”功能——只是不丢弃输入。已否决：仅内存的映射（随服务器消亡，且在早先分支中泄漏）。

### D7 — guides.show 今天是 CDN，明天是平台，基于 Cloudflare，独立部署
一个带静态资源的 Cloudflare Worker 在 `guides.show/v1/…` 下提供不可变的查看器文件。它与**粘贴服务毫无关系**（独立应用、独立部署、独立 wrangler 配置）。从第一天起预留 URL 空间：`/v1/…` 不可变资源、`/g/<id>` 未来共享引导、`/` 落地页。塑造它时，未来平台的效率、成本和规模是一等考量。

### D8 — 不可变、固定版本的查看器；语法和字体按需获取
`/v1/` 下的每个文件都带内容哈希，永不覆盖或删除，因此导出的 HTML 可无限期工作；HTML 固定确切的查看器构建（哈希 + `integrity`）。语法语法以每种语言一个小文件发布，导出器列出 diff 实际包含的语言，因此查看器只获取这些。字体使用真实字体（Inter + Geist Mono 拉丁子集，约 78 KB，已缓存），并带系统回退。基础 Plannotator 主题，现支持浅色/深色，可扩展到其他调色板。高亮在浏览器允许页面构造 worker 池时运行于 worker 池，否则在主线程——同一 bundle，一个运行时开关。worker 脚本从 guides.show 获取并在本地构造（blob 或 `data:` URL，classic 或 module），因为 `file://` 文档与 CDN 是不同源；可行的构造方式在每次页面加载时通过实时探测发现，因为浏览器行为不一致（Chrome 拒绝来自 `file://` 的 blob *module* worker，但接受 classic worker），且拒绝是异步的。4 秒后仍未初始化的池会被丢弃，引导在主线程重新渲染，因此死池绝不会让 diff 空白。

### D9 — 导出是纯函数，有三个调用方
`(snapshot) → HTML` 只存在于格式包中。调用方：(1) Plannotator UI 分享菜单 →“下载便携式引导”；(2) `plannotator guide export` CLI 子命令；(3) 日后，一个 agent 技能，自行生成引导 JSON 并通过 CLI 包装，让没有 Plannotator 应用的人也能获得引导。(1) 和 (2) 先发布；(3) 随后作为独立的 `plannotator-guide` 技能（后更名为 `guided-review`；独立仓库，不随 Plannotator 安装），通过 `plannotator guide export --guide guide.json --patch guide.patch` 实现——CLI 验证并包装，agent 只写引导。无需重新设计。

### D10 — 查看器发布随 Plannotator 发布
新的查看器构建由 GitHub Actions 工作流在 Plannotator 打标签发布时上传到 guides.show，受格式兼容性测试把关。由于资源是固定版本的（D8），糟糕的部署不会破坏现有导出。

### D11 — 分享链接不在范围内
不构建上传/`/g/<id>` 路由。分享菜单只显示“下载便携式引导”——没有禁用的占位项。一旦 guides.show 能托管，菜单预计会增加“创建分享链接”。

**附录（2026-08-15）：已构建。** 分享链接现已存在，遵循 `adr/implementation/guide-share-hosting.md` 中的契约（范围备忘录：`adr/research/scope-guide-share-hosting-20260815.md`）。已决定并交付的内容：

- 默认加密：上传者使用与计划分享链接相同的 `@plannotator/core` 原语压缩并加密快照，密钥只存在于 URL 片段（`#key=`）中，因此宿主持有无法读取的密文，提供的页面在浏览器中解密。`--public` /“允许链接预览”是选择加入项，以未加密方式存储快照，使页面可携带标题和 `og:` 标签。
- 默认无过期；`ttl` 是每个分享的可选项，也是唯一的过期机制（宿主不施加任何过期）。一次性删除令牌可移除引导；Plannotator 在已保存引导上记录链接和令牌（每个引导一个链接），以便应用日后移除它。
- 服务位于 `apps/guides-show`（`share/` 纯处理器 + R2 和内存存储），仅由 Cloudflare Worker 通过 R2 桶提供（`POST /api/g`、`GET /g/<id>`、`GET /api/g/<id>`、`DELETE /api/g/<id>`）。自托管意味着在你自己的 Cloudflare 账户下部署该 Worker；没有其他宿主目标，且与粘贴服务一样，唯一的上传防护是 25 MiB 大小上限（无限流，无运营方生命周期上限）。CLI（`plannotator guide share` / `unshare`）无需运行中的 Plannotator 即可分享。`PLANNOTATOR_GUIDE_SHARE_URL` / `guideShareUrl` 指向你自己的部署；`PLANNOTATOR_SHARE=disabled` 将其关闭。
- 托管页面带有托管元信息和客户端下载按钮，可重建便携式文件，因此链接绝不会把引导困在宿主上。可下载文件仍无上限（D1）；宿主将存储正文限制在 25 MiB。
- 将 Worker 与 `guides-show-guides` 桶部署到生产是单独的显式步骤，不属于构建的一部分。

### D12 — 现在不做（明确）
导出中的标注；PR 评论线程；基础主题之外的调色板；面向商业平台的任何嵌入组件 API（平台若需要引导，就提供同一 HTML 页面）。分享链接于 2026-08-15 离开此列表（见 D11 附录）。仍不在范围内：guides.show 上的列表或账户、从托管页面提交反馈、默认过期。

## 影响

- `packages/review-editor` 将引导链让给 `packages/guide-viewer` 并重新导入；应用行为必须不变（一致性门禁）。
- 约 6 个仅类型模块从 `packages/shared → packages/core` 移动，使查看器没有 Node 导入。
- 查看器需要自己的 Tailwind 构建（现有 `@plannotator/ui/styles.css` 不扫描 review-editor 类）。
- npm 包的使用者必须将 `shiki` 别名到收窄的语法入口（如应用已对 `shiki/wasm` 所做），否则要付出约 11.6 MB 的代价——已记录为硬性要求。
- Bun 和 Pi 服务器获得相同的导出端点；Pi 内置格式包。

## 参考
- `adr/implementation/portable-guided-reviews.md` — 实现规范、阶段、验证矩阵
- `adr/implementation/guide-share-hosting.md` — 分享链接托管契约（D11 附录）
- `adr/decisions/006-guided-review-first-class-feature-20260702-192821.md`
- `adr/research/SPIKE-guide-diff-annotation-reuse-20260702-194831.md`
- 分支 `guided-review-html-export`（`2cf18cc3`）— 保留来源