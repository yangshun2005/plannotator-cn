# 范围：在 guides.show 上托管共享指南

日期：2026-08-15
状态：范围界定中——已提出决策，尚未构建。承接 ADR 007（D7 预留了 URL 空间，D11 将分享链接排除在范围之外）。

> **结果（2026-08-15）。** 按提案构建，随后简化。最终交付：分享服务位于 `apps/guides-show/share/`（纯处理器 + `GuideStore` 接口，R2 和内存存储），由现有 Cloudflare Worker 通过 `guides-show-guides` R2 桶提供服务；默认加密链接，可选 `--public`；默认无过期，可选每次上传的 `ttlSeconds`；一次性删除令牌；`plannotator guide share` / `unshare` CLI；Plannotator 中的分享菜单；`PLANNOTATOR_GUIDE_SHARE_URL` / `guideShareUrl`。被砍掉的内容：Bun 自托管目标以及 fs 和 S3 存储（及其 `--trust-proxy` / `--public-origin` 标志）、上传速率限制、运营方 TTL 上限、CLI 的 `--service-url` 标志。原因：一个托管目标足以让路由保持诚实和可测试，而每条额外路径都是没有用户的代码。"可自托管"现在意味着在你自己的 Cloudflare 账户下部署同一个 Worker；唯一的上传防护是大小上限，与粘贴服务相同。本备忘录其余部分是构建前的提案，保持原样。契约见 `adr/implementation/guide-share-hosting.md`。

## 摘要

如今可移植指南是一个文件。下一步是一条链接：`https://guides.show/g/<id>`。几乎所有需要的东西都已存在——产物格式（基于严格快照的 `createGuideHtml`）、`/v1/` 下固定不可变的查看器、该域名上预留了 `/g/*` 和 `/api/*` 的 Worker，以及端到端加密的分享模式（计划分享链接：密文在粘贴服务中，密钥在 URL 片段中）。分享服务是其上的一层薄封装：存储快照，发放能力 URL，用固定查看器渲染。

建议**在 `apps/guides-show` 内**构建，采用粘贴服务的形态（`core/` 处理器 + 存储接口，`stores/` r2|fs|s3，`targets/` cloudflare|bun），**Cloudflare 优先，通过配置可自托管**，**默认加密**链接以匹配 Plannotator 现有的计划分享方式，并为未来平台保留明文（服务器可读）模式。Plannotator 在"下载可移植指南"旁新增"创建分享链接"、`plannotator guide share` CLI，以及一个指向自托管源的环境变量。技能仓库仍保持为技能。

## 现有内容

| 组件 | 状态 | 复用 |
|---|---|---|
| 快照格式 + `createGuideHtml` | 已发布，`@plannotator/core/guide-format`（浏览器安全，可在 Worker 中运行） | 我们存储和渲染的对象 |
| `guides.show/v1/<hash>` 处的查看器 | 在线，仅追加，SRI 固定 | `/g/<id>` 用固定构建渲染，与文件相同 |
| Worker `apps/guides-show/worker` | 在线；`/g/*`、`/api/*` 已预留（404 `{ reserved: true }`） | 成为宿主 |
| 粘贴服务 `apps/paste-service` | 已发布：`core/handler.ts` + `storage.ts` 接口，`stores/{fs,kv}`，`targets/{bun,cloudflare}`，5 MB 上限，7 天 TTL，8 字符 id | 架构模板（不是代码：指南更大、更长寿、需要 HTML 渲染） |
| 计划分享链接 | 端到端加密，`@plannotator/core/crypto`，密钥在 `#key=` 中，`PLANNOTATOR_SHARE_URL` / `PLANNOTATOR_PASTE_URL` / `PLANNOTATOR_SHARE=disabled` | 隐私模型 + 配置先例 |
| 生产者 | UI 按钮、`guide export --id`、`guide export --guide/--patch`（技能） | 各自获得一个 `share` 同级命令 |

## 待做决策

### S1 — 服务所在位置
**建议：`apps/guides-show`**，紧邻查看器构建和它所扩展的 Worker。一次部署、一个域名、一份清单。否决：技能仓库 `plannotator/guides`（`npx skills add` 会克隆仓库；服务放在那里会混淆安装，并引入查看器所需的 monorepo 包）；第三个仓库（查看器构建依赖 `packages/*`；在出现单独拥有托管职责的团队之前没有收益）。

### S2 — 存储内容
**建议：快照 JSON，加上上传时记录的查看器固定版本。** `/g/<id>` 在读取时用 `createGuideHtml`（纯函数，已 Worker 安全）渲染 HTML 并在边缘缓存。与下载得到的字节相同——无漂移（D9）。存储快照而非 HTML 为后续保留 JSON 接口（`/g/<id>.json`、嵌入、平台），并让链接将来可选择更新的查看器而无需重新上传。存储固定版本，使其今天不会静默改变。

### S3 — 隐私模型
**建议：默认加密，显式选择明文。**
- *加密*：客户端（Plannotator 服务器或 CLI）用 `@plannotator/core/crypto` 加密快照，上传密文，获得 `/g/<id>#key=…`。`/g/<id>` 提供查看器外壳，获取 `/api/g/<id>`，在浏览器中解密、解析、渲染。宿主永远看不到代码。这正是用户已有的计划分享契约；diff 比计划更敏感，而非更少。代价：无服务器端标题/OG 预览（通用卡片），无服务器端内容功能。
- *明文*：快照以可读形式存储；`/g/<id>` 是完整 HTML，标题/意图在 OG 标签中；为后续的列表、评论、嵌入提供支持。上传时选择加入（`--public` / 复选框），绝不默认。
- 两者都是不公开的能力 URL：128 位 id（22 个 base64url 字符），而非粘贴服务的 8 字符。通过上传时返回一次的 `deleteToken` 删除（存储哈希值）。

### S4 — 限制与生命周期
- 托管大小上限（文件没有——D1——但宿主必须有）：**25 MB** 密文/快照；超过则提示"改为下载"并显示大小。
- 生命周期：指南**默认无过期**（7 天就失效的评审链接比文件更糟）；为偏执者提供可选 TTL 参数。通过令牌按请求删除。当存在带账户的平台时重新考虑。
- 滥用：上传的每 IP 速率限制（Cloudflare 速率限制规则或 Durable Object 计数器）、大小上限、`Content-Type` 允许列表。仅在需要时使用 Turnstile。

### S5 — 自托管
与粘贴服务形态相同，记录在 `apps/guides-show/README.md`：
- **Cloudflare（一等）**：`wrangler deploy`，用 R2 桶存指南（KV 的 25 MB 值上限和最终一致性使 R2 成为正确存储；查看器已在 R2 中）。通过 `wrangler.toml` 变量配置。
- **Bun 目标**：`bun run apps/guides-show/targets/bun.ts` 配合 `stores/fs`（或 `stores/s3`），用于公司 VPN 后的机器。也从 `dist/viewer` 提供 `/v1/` 查看器资源，使自托管源自身完整。
- Plannotator 侧：`PLANNOTATOR_GUIDE_SHARE_URL`（默认 `https://guides.show`）——镜像 `PLANNOTATOR_SHARE_URL`；`PLANNOTATOR_SHARE=disabled` 随其他一切关闭指南分享；`PLANNOTATOR_GUIDE_VIEWER_URL` 已覆盖自托管查看器。配置文件等价项在 `~/.plannotator/config.json`。

### S6 — Plannotator 集成
- **UI**：指南屏幕上的分享菜单在下载旁新增**创建分享链接**（D11 已预见）。流程：点击 → 确认对话框（上传什么、加密、不公开、大小）→ 评审服务器上的 `POST /api/guide/:id/share` → 服务器加密 + 上传 → 对话框显示 URL、复制按钮，以及仅显示一次的删除令牌。未经确认绝不上传（与大型计划分享规则相同）。
- **服务器**：`POST /api/guide/:id/share`（Bun + Pi 镜像）、使用存储令牌的 `DELETE` 对应端点，以及指南信封中每指南的 `{ url, createdAt }` 记录，使已保存的指南记住其链接。
- **CLI**：`plannotator guide share --id <saved> | --guide g.json --patch p.patch [--public] [--ttl]` 打印 URL（删除令牌到 stderr）。`plannotator-guide` 技能（后更名为 `guided-review`）获得一行可选说明："如果用户想要链接，运行 `guide share` 而非 `guide export`"。
- **查看器**：`/g/<id>` 与 `/v1/` 同源，因此 worker 池走普通模块 worker 路径；探测已处理。在托管页面添加"下载此指南"入口（文件是可移植形式）。

### S7 — 预留，不构建
账户、列表/搜索、共享指南上的评论或标注（格式槽已存在，D3）、嵌入/iframe API、团队自定义域名。URL 空间（`/g/`、`/api/`）和静态 JSON 选择（S2）使所有这些保持开放。

## 架构（提案）

```
apps/guides-show/
  worker/index.ts          现有：/v1 不可变、落地页、→ 挂载分享路由
  share/
    core/handler.ts        纯请求处理：创建/获取/删除、限制、id 生成、通过 createGuideHtml 渲染 HTML
    core/storage.ts        接口 { put(id, blob, meta), get(id), delete(id) }
    stores/r2.ts  fs.ts  s3.ts
    targets/cloudflare.ts  绑定 R2 + 速率限制；targets/bun.ts 从磁盘提供分享 + /v1
  site/                    落地页（不变），后续："粘贴指南"上传页？（非 v1）
```

路由：
- `POST /api/g` — 请求体 `{ mode: "encrypted"|"plain", viewer: <manifest pin>, data }` → `{ id, url, deleteToken }`
- `GET  /g/<id>` — 明文：完整 HTML（OG 标签，`Cache-Control: public, s-maxage=…`，删除时清除）；加密：查看器外壳，获取 `/api/g/<id>` 并用 `#key` 解密
- `GET  /api/g/<id>` — 原始存储载荷（JSON 或密文）；`GET /g/<id>.html` — 可移植下载
- `DELETE /api/g/<id>` — 带 `Authorization: Bearer <deleteToken>`

## 阶段

1. **服务** — `share/core` + r2 存储 + Worker 路由，加密 + 明文模式，限制，按粘贴服务测试形态的测试；部署；冒烟：上传一个夹具，从另一源打开 `/g/<id>`。
2. **Plannotator** — 服务器端点（两个运行时），分享菜单"创建分享链接"带确认对话框，`guide share` CLI，环境/配置接线，文档（`portable-guides` 页面新增"托管链接"），遵守 `PLANNOTATOR_SHARE=disabled`。
3. **自托管** — Bun 目标 + fs/s3 存储，README 配方，`PLANNOTATOR_GUIDE_SHARE_URL` 对本地实例端到端。
4. **技能** — 一旦发布版本包含 `guide share`，在 `plannotator/guides` SKILL.md 中添加一个可选步骤。

## 开放问题（需要你决定）

1. 默认加密（S3）？它匹配计划分享且是更安全的产品，但代价是链接预览，并在用户选择明文前阻止服务器端功能。
2. 过期（S4）：默认无，还是长默认（如 1 年）以保持桶的整洁？
3. 托管链接应从 Plannotator **服务器**创建（从用户机器上传，需要评审服务器运行），还是 CLI 也应完全离线于 Plannotator 工作（`guide share --guide/--patch`）？建议：两者，共享一个函数。