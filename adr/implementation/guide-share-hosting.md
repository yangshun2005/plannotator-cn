# 指南分享托管 — 实现契约

日期：2026-08-15。范围备忘录：`adr/research/scope-guide-share-hosting-20260815.md`。其中已确定并确认的决策：默认加密，通过 `--public` 选择明文；默认不过期（`ttl` 可选）；CLI 分享无需运行 Plannotator；服务位于 `apps/guides-show`；仅支持 Cloudflare — 自托管意味着在你自己的账户下部署同一个 Worker。

本文档是各部分编码所依据的契约。以下名称均为最终版；如需更改，请先在此处修改。

## 1. 载荷格式

两种模式。两者都为每个指南存储一个文本正文。

| 模式 | 存储的正文 | 如何生成（上传方） | 如何消费（查看方） |
|---|---|---|---|
| `encrypted`（默认） | `encrypt(await compress(snapshot))` — 一个 base64url 字符串（`@plannotator/core/crypto` 的 `encrypt` 作用于 `@plannotator/core/compress` 的 `compress`，与计划分享链接完全一致） | 上传方保留 `key`，将其放入 URL 片段 `#key=<key>`（`GUIDE_SHARE_KEY_PARAM`） | 查看方获取正文，`decrypt(body, key)` → `decompress(...)` → `parseGuideSnapshot(value)` |
| `plain` | 快照 JSON 文本（服务端必须通过 `parseGuideSnapshotJson`） | `--public` / "允许链接预览" | 服务端渲染 `createGuideHtml(snapshot, { viewer, hosted })`；查看方像现在一样读取内嵌快照 |

`snapshot` 是来自 `@plannotator/core/guide-format` 的 `GuideSnapshotV1`（`buildGuideSnapshot` / `buildSavedGuideSnapshot` / `buildAuthoredGuideSnapshot`）。

存储正文的大小上限：`MAX_SHARED_GUIDE_BYTES = 25 * 1024 * 1024`。超过上限 → `413 { error: "too large", maxBytes }`。（可下载文件没有上限 — D1；托管方必须有。）

## 2. 标识符与令牌

- `id`：16 个随机字节，base64url（22 个字符）。未列出的能力 URL。
- `deleteToken`：16 个随机字节，base64url，创建时仅返回一次；以十六进制 SHA-256 存储（`deleteTokenHash`）。
- 查看方固定：上传方发送其在导出中内嵌的 `{ js, css, jsIntegrity, cssIntegrity, langs? }`（其 `GUIDE_VIEWER_MANIFEST`）；服务端存储并用它渲染。若缺失，服务端使用自己捆绑的清单。`baseUrl` 从不发送 — 托管方始终使用自己的查看方基础地址（`https://<host>/v1/`，即 Worker 自身的源）。

## 3. 存储接口（`apps/guides-show/share/core/storage.ts`）

```ts
export interface StoredGuideMeta {
  readonly mode: 'encrypted' | 'plain';
  readonly createdAt: string;            // ISO
  readonly expiresAt?: string;           // ISO, absent = never
  readonly deleteTokenHash: string;      // hex sha256
  readonly viewer?: { js: string; css: string; jsIntegrity?: string; cssIntegrity?: string; langs?: Record<string, string> };
  readonly bytes: number;                // body length
}
export interface GuideStore {
  put(id: string, body: string, meta: StoredGuideMeta): Promise<void>;
  get(id: string): Promise<{ body: string; meta: StoredGuideMeta } | null>;   // null when missing OR expired (store may lazily delete)
  delete(id: string): Promise<void>;
}
```
存储实现：`stores/r2.ts`（对象 `g/<id>` 正文 + `g/<id>.meta` JSON；meta 对象先写入、最后删除，因此写了一半的指南会被读取为不存在）、`stores/memory.ts`（测试用）。

## 4. HTTP API（Worker）

| 路由 | 请求 | 响应 |
|---|---|---|
| `POST /api/g` | JSON `{ mode: 'encrypted'\|'plain', data: string, viewer?: {…}, ttlSeconds?: number }` | `201 { id, url, deleteToken, expiresAt? }`，其中 `url = <origin>/g/<id>`（加密模式下上传方自行追加 `#key=…`）。错误：`400` 格式错误 / 明文未通过 `parseGuideSnapshotJson`（附带 `path`+`message`），`413` 超过上限，`429 { error: 'too many requests' }` + `Retry-After` 超过每 IP 创建限制。 |
| `GET /g/<id>` | — | 明文：`200 text/html`，来自 `createGuideHtml(snapshot, { viewer, hosted: { url } })`；加密：`200 text/html`，来自 `createGuideShellHtml({ viewer, hosted: { url }, payloadUrl: '/api/g/<id>' })`。`Cache-Control: public, max-age=300`。`404` 未知/已过期：一个小型 HTML "not found" 页面（与回退页相同的样式类）。 |
| `GET /api/g/<id>` | — | 存储的正文：加密 → `text/plain; charset=utf-8`；明文 → `application/json`。`Access-Control-Allow-Origin: *`，`Cache-Control: public, max-age=300`。`404 { error: 'not found' }`。 |
| `DELETE /api/g/<id>` | `Authorization: Bearer <deleteToken>` | `204`；`401` 令牌错误/缺失；`404`。 |
| `OPTIONS /api/g*` | — | CORS 预检（`*`，`POST, GET, DELETE, OPTIONS`，`Content-Type, Authorization`）。 |
| `GET /healthz` | — | 不变。 |

匿名上传有两道刹车，均在代码中：大小上限，以及**仅针对创建**的速率限制。`POST /api/g` 经过 Cloudflare 原生速率限制绑定（`wrangler.toml`：`[[ratelimits]] name = "GUIDE_CREATE_LIMITER"`，`simple.limit = 20`，`simple.period = 60`；运行时只接受 10 或 60 的周期，其他一律不接受），以 `CF-Connecting-IP` 为键，返回 `429 { error: 'too many requests' }` 并附带 `Retry-After: 60`。上限刻意宽松：一个人一小时分享几次指南，因此每分钟 20 次只有脚本才会触及，合规用户不会察觉。读取、`DELETE`（已由创建时发放的删除令牌把关）、预检和 `/v1/` 不受限制。该绑定在 Worker 的 `Env` 中是 OPTIONAL，且在所有无法解析的方向上检查均失败开放（无绑定，因此 `wrangler dev`、去掉该块的自托管和测试均不受限；无 `CF-Connecting-IP`，因此 Worker 前面没有可作键的东西；绑定抛出异常）。该刹车保护托管部署，绝不能成为破坏本地运行或自托管的原因。

保留期是决策而非疏忽：没有运营方生命周期上限，没有任何东西清理桶，因为一个定时死亡的评审链接是比文件更差的产品。（Plannotator 的粘贴服务是匿名、未认证上传的先例，但其防护更严格：5 MB 和 7 天 TTL。）过期是按上传的（`ttlSeconds`），从不由托管方强加，且在读取时执行：过期的指南在下次任何路由触及时被删除，因此无人打开的过期指南会留在桶中。值得在精简分享重构时重新审视，而非之前。

存储失败返回 `500 { error: 'internal error' }`（API）或样式化错误页；存储自身的消息记录在托管方侧（`GuideShareContext.logError`，默认 `console.error`），从不返回。错误页标明服务主机（`url.host`），而非 guides.show，并复用 core 导出的 `FALLBACK_STYLE`（`.pgr-fallback`）。

Worker `wrangler.toml`：添加 `[[r2_buckets]] binding = "GUIDES" bucket_name = "guides-show-guides"`（桶必须在部署前创建 — 部署是单独的、后续步骤；不作为本次构建的一部分完成）。

## 5. Core 新增（已完成）

`packages/core/guide-format.ts` 现在导出：`GuideHtmlOptions.hosted?: { url }`、`GuideHostedPage`、`createGuideShellHtml({ viewer, hosted, payloadUrl })`、`GUIDE_HOSTED_META_NAME`、`GUIDE_PAYLOAD_META_NAME`、`GUIDE_SHARE_KEY_PARAM`。托管页面携带 `<link rel=canonical>`、`og:*`、`robots noindex` 和托管 meta；当托管源与查看方源不同时，CSP `connect-src` 包含托管源。通过 `@plannotator/shared/guide-format` 重新导出；Pi 通过 `vendor.sh` 引入。

## 6. 查看方（`apps/guides-show/viewer/main.tsx`）

启动：若存在 `#plannotator-guided-review` 脚本 → 走当前路径。否则若存在 `<meta name=GUIDE_PAYLOAD_META_NAME>` → 托管加密：从 `location.hash` 读取 `key`（对片段使用 `URLSearchParams`），获取 `content` URL，`decrypt` → `decompress` → `parseGuideSnapshot`。错误卡片：缺少密钥（"此链接缺少密钥 — `#` 之后的部分"）、404 / 网络（"此指南已不可用"）、解密失败（"此链接中的密钥无法打开此指南"）。获取时渲染骨架。

托管页面（任一模式，通过 `<meta name=GUIDE_HOSTED_META_NAME>` 检测）：`headerActions` 在模式切换旁增加一个 **Download** 按钮，在客户端构建可移植文件 — `createGuideHtml(snapshot, { viewer })`，其中 `viewer` 从 DOM 重建（样式表 `href`+`integrity`，模块脚本 `src`+`integrity`；`baseUrl` = 脚本所在目录；无 `langs`）— 并通过 blob URL（`<a download>`）保存；文件名 `guideExportFilename(title)`。从不重新从服务端获取；从不包含托管 meta（该文件是纯导出）。

## 7. Plannotator 生产者

- `packages/server/guide/guide-share.ts`（Bun；像 `guide-review.ts` 一样引入到 Pi）：`shareGuide(snapshot, opts: { serviceUrl: string; mode: 'encrypted'|'plain'; ttlSeconds?: number; viewer: Omit<GuideViewerAssets,'baseUrl'>; fetch?: typeof fetch }) → Promise<{ id: string; url: string; deleteToken: string; expiresAt?: string; bytes: number }>`，其中加密模式下 `url` 包含 `#key=…`。错误以 `GuideShareError` 抛出，带 `{ status?, message }`（服务不可达、413、4xx/5xx 带正文消息）。
- 配置（`packages/shared/config.ts`）：`resolveGuideShareUrl(config)`：环境变量 `PLANNOTATOR_GUIDE_SHARE_URL` → `config.guideShareUrl` → `https://guides.show`；必须是 `http(s)`；去除尾部斜杠；无效 → 警告一次，使用默认值。当 `resolveSharingEnabled(config)` 为 false（`PLANNOTATOR_SHARE=disabled` / `{ "share": "disabled" }`）时，分享完全禁用。
- 评审服务端点（Bun `packages/server/review.ts` 和 Pi `apps/pi-extension/server/serverReview.ts`，两者）：
  - `POST /api/guide/:jobId/share` 正文 `{ public?: boolean, ttlSeconds?: number }`（所有字段可选；空正文表示默认值）→ `200 { id, url, deleteToken, expiresAt?, bytes, recorded }`；`403 { error: 'sharing disabled' }`；`404` 当指南不可导出时（与 `/export` 相同的解析）；`409 { error, url }` 当指南的信封已记录链接时（每个指南一个链接：记录是令牌唯一存放处，因此第二次上传会使第一个成为孤儿；请先移除）；`502 { error }` 服务失败时。与其他变更端点相同的同源守卫。成功时，已保存的指南信封（当存在 / 在 `saved:` id 之后）记录 `share: { id, url, createdAt, deleteToken, serviceUrl }`，以便已保存的指南记住其链接及其所在主机；`recorded` 表示是否发生（当没有信封时为 false，例如指南历史关闭），因此 UI 只对可移除的链接提供 Remove link。
  - `DELETE /api/guide/:jobId/share` → 用存储的令牌调用记录所指的主机（`serviceUrl`，否则为记录的 `url` 的源；绝不只调用当前配置的分享 URL），清除信封记录；`204`；`404` 无记录。主机 `404`（已不存在）仍清除记录。
  - `GET /api/guide/:jobId/share-info` → `{ enabled: boolean, serviceUrl, existing?: { url, createdAt } }`。
- CLI（`packages/server/guide/guide-cli.ts`）：`plannotator guide share --id <saved> | --guide g.json --patch p.patch | --snapshot s.json [--public] [--ttl <e.g. 7d|24h|3600>] [--json]`。stdout：URL（或带 `--json` 的 `{ id, url, deleteToken, expiresAt? }`）；stderr：`Delete with: plannotator guide unshare <id> --token <deleteToken>` 和大小。当已保存的指南已记录链接时，`--id` 拒绝（退出码 1，不上传），打印该链接及其 unshare 命令。`plannotator guide unshare <id> --token <t>` → `204` → "Removed"；某个已保存指南记住的 id 会从其记录所指的主机移除。退出码同 `export`：0 / 1（未找到、无效、服务错误）/ 2 用法。当 `PLANNOTATOR_SHARE=disabled` 时拒绝（退出码 1）。Hook 入口点将 `guide` 之后的原始 argv 交给指南 CLI（它从其他所有内容中剥离 annotate 门控标志，包括 `--json`）。
- UI（`packages/review-editor/components/guide/GuideExportButton.tsx` → 变为 Share 菜单）：**Download portable guide**（不变）和 **Create share link**（当 `/share-info` 显示禁用时隐藏）。对话框：将上传的内容（指南 + 差异，大小来自 `/export-info`），"端到端加密：guides.show 永远看不到代码；密钥在链接中"（默认）带复选框"允许链接预览（以未加密方式存储指南）" = `public`；Create → 显示 URL 和 Copy 按钮，以及仅显示一次的删除令牌及其自己的 Copy；若 `existing`，显示链接和 **Remove link**。不点击不上传。

## 8. 自托管

在你自己的 Cloudflare 账户下部署同一个 Worker：创建两个 R2 桶，`bun run build:viewer && bun run deploy:viewer`，`wrangler deploy`（README "Self-hosting"）。没有其他托管目标。

## 9. 视口优化

`GuideViewportProvider` 接受 `eager?: boolean`；为 true 时每个注册的 shell 都挂载（无窗口）。`GuideView` 设置 `eager={files.length <= GUIDE_EAGER_MOUNT_MAX_FILES}`（`= 15`）。适用于应用内和可移植。

## 10. 必须存在的测试

- `share/core/handler.test.ts`：使用内存存储对两种模式进行创建/获取/删除；上限；错误的明文快照被拒绝并带路径；删除令牌语义；过期被遵守；明文 HTML 有 og:title + 托管 meta 且无载荷 meta；加密 shell 有载荷 meta 且无 title/intent。
- `worker/index.test.ts`：仅接线 — 分享路由通过 `worker.fetch` 在伪造的 `GUIDES` 桶上分发，以捆绑清单作为回退固定；`/v1` 资产头；404 页面；通过环境注入的伪造绑定测试创建速率限制（耗尽时 429 + `Retry-After`，按 IP 作键，读取/删除/预检/资产从不查询，无绑定、无客户端 IP 或限制器抛出时失败开放）。
- `packages/server/guide/guide-share.test.ts`：`shareGuide` 针对伪造 fetch（加密 → url 有 `#key=`，载荷解密回快照；明文发送 JSON；错误映射）。
- `guide-cli.test.ts`：`share`/`unshare` 使用打桩服务（注入 `fetch`），退出码，`--json`。
- `guide-persistence.test.ts`（或新文件）两个运行时：`/share`、`/share-info`、`DELETE`、分享禁用 403、信封记录。
- 查看方：启动路径没有 DOM 测试框架 — 由下面的手动/无头冒烟覆盖。
- `GuideViewportManager` eager 模式：eager 时所有 shell 挂载，否则保留窗口。

## 11. 冒烟（构建结束时；不部署生产）

`bun run --cwd apps/guides-show deploy:viewer -- --local && bun run --cwd apps/guides-show dev:worker` + `PLANNOTATOR_GUIDE_SHARE_URL=http://localhost:8787 PLANNOTATOR_GUIDE_VIEWER_URL=http://localhost:8787/v1/ plannotator guide share --guide … --patch …` → 在无头 Chrome 中打开打印的 URL：指南渲染（加密路径），Download