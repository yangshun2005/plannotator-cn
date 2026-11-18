# 手机上的 Plannotator Inbox：通信契约

日期：2026-10-08。移动端计划的 C1 步骤。本文遵循的记录：所有者的十三项移动端决策、iPhone 设计记录（渲染图 1.1 至 9.2）以及构建计划。这些设计记录私下保存，不在本仓库中；构建所需的行为写在这里，裸渲染图编号（1.3、9.2）指代那份私有记录。

本文档是 iPhone 应用（以及第 10 节所述的 Android 应用）、Inbox 的设备入口、LAN 监听器、中继 Worker 和界面桥接所依据的契约：P1、P2、S1、R1、R2 和 M1 至 M6。以下名称均为最终名称。后续变更先修改本文件。裸编号如 1.3 或 4.3 指代 iPhone 记录的渲染图；"exchange 7.x" 和 Exchange 列指代第 7 节。

总体姿态，一段话概括。Inbox 保持为回环服务器。手机配对一次，获得自己的随机 bearer token。三条路径承载相同的请求：tailnet（在回环端口前运行 `tailscale serve`）、同一 Wi-Fi（TLS 监听器，手机固定其证书）和中继（Cloudflare Worker，仅持有配对时生成的密钥下的密文）。每个手机请求都经过一个入口 `/api/inbox/device/*`，它将显式允许列表映射到窗口现有的处理器。没有其他东西响应手机。

存在两项限制，因为配对离不开它们：一个 offer 存活 10 分钟，其六位数字码在 5 次错误尝试后关闭。中继的邮箱创建复用 guides.show 已在其创建路由上使用的数字，每 IP 每分钟 20 次（第 4 节）；由于 Cloudflare 的速率限制绑定是近似且按机器计数的，一个按 IP 的 Durable Object 在其后持有硬计数。Apple 自身的上限在 Apple 设定之处适用（推送体 4096 字节，四个通知操作）。没有其他限制。

## 1. 配对 offer 及其 QR 载荷

### 创建 offer

`POST /api/inbox/pairing` 是窗口路由，带有窗口的守卫（Host 允许列表、`isSameOriginOrNoOrigin`、`serverSession` 过期标签页检查）。请求体 `{ serverSession? }`。返回 `201` 及 offer 和窗口绘制为 QR 码的链接（`uqr`，已是 `packages/server` 的依赖）：

| 字段 | 含义 |
|---|---|
| `offer.code` | 六位十进制数字，在 `000000` 到 `999999` 上均匀分布，显示在 QR 码旁供手动输入（1.2、1.3） |
| `offer.expires_at` | ISO 时间，offer 创建后 10 分钟 |
| `link` | 下方的 `plannotator://pair` 链接 |
| `computer.name` | 计算机名称：macOS 用 `scutil --get ComputerName`，否则用 `os.hostname()` 的第一个标签 |
| `addresses.tailnet` | tailnet 发布的 `host:port`，当"从我的 tailnet 访问"关闭时为 null |
| `addresses.lan` | LAN 监听器的 `ip:port`，当"从此 Wi-Fi 访问"关闭时为 null |
| `addresses.fingerprint` | LAN 证书的 SHA-256（第 3 节），LAN 关闭时为 null |
| `reachable` | 该码携带手机可用的地址（2026-10-12 为窗口的"配对手机"流程添加，该流程仅在某种方式开启后才显示码；手机从不读取此应答） |

窗口还读取以下内容，用于"配对手机"（窗口路由，从不在入口上；手机的交换不变）：`GET /api/inbox/pairing`，最后一个 offer 的状态及其显示期间到达入口的内容（`attempts`：哪个监听器，以及错误码、Wi-Fi 上的数字、旧码、关闭后、或已兑换；从不包含密钥或码），以及 `GET /api/inbox/pairing/checks`，此计算机在任何方式开启前对两种方式能告知的内容。

一个 offer 仅可使用一次。它在被兑换、过期、5 次错误码后、或窗口创建新 offer 时关闭：同一时间只有一个 offer 开放，因此六位数字始终指向一个 offer。offer 仅存在于服务器内存中；重启即关闭。

配对需要手机通过同一 Wi-Fi 或 tailnet 访问计算机一次；中继在 v1 中不承载兑换。两个地址均为 null 时 offer 仍会应答（模拟器通过 `127.0.0.1` 访问 Inbox，`reachable: false`），窗口从不显示这样的码："配对手机"从"你的手机将如何访问此计算机？"开始，仅在某种方式开启后才显示码。

### tailnet 发布

"从我的 tailnet 访问"打开第二个回环监听器，仅服务入口而不服务其他内容（第 2 节"入口应答之处"），并通过 `tailscale serve --bg --https=8443 http://127.0.0.1:<door port>` 发布：用 serve，绝不用 funnel。窗口的端口从不发布。HTTPS 端口为 8443，保存在 `inbox.json` 中为 `tailnet: { https_port: 8443, door_port }`，因此手机持有的地址在本地端口变动后仍有效；入口监听器在 `door_port` 空闲时再次使用它，映射在每次启动时指向它。当另一个 serve 映射已占用 8443 时，窗口会说明，开关保持关闭：Inbox 从不覆盖非自己创建的映射。`buildServeArgs` 在 P1 中获得此双端口形式。

当映射的代理目标是本次运行的入口监听器或上次运行的（`door_port`）时，该映射属于 Inbox；只有该映射会被重新指向或拆除。映射和入口监听器在开关关闭时以及每次干净停止时拆除（退出、停止路由、以及为更新而重启，后者在启动新二进制前拆除它们，因此两者永不竞争）。开关在 `inbox.json` 中保持开启，下次启动再次发布。崩溃遗留的映射（kill -9）由下次启动在任何其他操作前重新指向，或在该启动无法发布时拆除；前台 Inbox 的终端挂断（SIGHUP）在映射存在时执行干净停止；`uninstall --purge` 通过 `door_port` 拆除遗留映射。拆除失败（Tailscale 无法应答）不改变任何内容：此时关闭开关返回 `409 tailnet_unavailable` 及手动命令，`door_port` 保留在 `inbox.json` 中供下次尝试。窗口自身的端口从不将 tailnet 名称加入其 Host 允许列表（在 P1 后续中添加）。

开关的窗口路由（在 P1 中添加；初稿只命名了开关但无路由）：

- `GET /api/inbox/tailnet`：`{ tailnet: { on, address, error } }`。`on` 是 `inbox.json` 保存的人员开关；`address` 是已发布的 `host:port`，发布不工作时为 null；`error` 用窗口的措辞说明原因（例如启动时 Tailscale 停止或登出）。
- `POST /api/inbox/tailnet`（窗口守卫）`{ serverSession?, on }`：`{ tailnet }`。开启时立即发布：当另一个映射占用 8443 时 `409 tailnet_port_taken`，当 Tailscale 无法发布时（未安装、未运行、已登出）`409 tailnet_unavailable`，开关保持关闭。关闭时拆除 Inbox 自己的映射并从 `inbox.json` 清除 `tailnet`。

手机如何在 tailnet 上找到计算机（1.3"在你的 tailnet 上"）：它不发现。Bonjour 不跨越 tailnet，iOS 应用无法向 Tailscale 查询其对等节点，正如记录中 1.3 的说明所述。该节列出此手机已知的每台计算机的 tailnet 地址（来自扫描的 QR）以及一行用于输入 Inbox 打印的 `host:port`；无论哪种方式，人员随后输入六位数字。

### 链接

```
plannotator://pair?v=1&name=MacBook%20Pro&tailnet=macbook-pro.tail0000.ts.net%3A8443&lan=192.168.1.24%3A47321&fp=abab...ab&secret=AAAA...A&code=482913
```

| 参数 | 值 | 出现条件 |
|---|---|---|
| `v` | `1` | 始终 |
| `name` | `computer.name`，百分号编码 | 始终 |
| `tailnet` | `host:port`，HTTPS | tailnet 发布开启时 |
| `lan` | `ip:port`，HTTPS 及固定证书 | LAN 监听器开启时 |
| `fp` | LAN 证书的 SHA-256，64 个小写十六进制 | 与 `lan` 一起 |
| `secret` | 配对密钥：32 个随机字节，base64url 无填充（43 字符） | 始终 |
| `code` | 六位数字 | 始终 |

手机将未知 `v` 的链接读作"更新应用"。两个地址按顺序尝试（先 LAN，后 tailnet）以 QR 的密钥进行兑换；手动输入的数字仅发往 tailnet 或回环，绝不发往 LAN（第 3 节）。中继的地址和邮箱在兑换的应答中到达，不在链接中。

## 2. 设备令牌入口

### 设备记录

`inbox/devices.jsonl`，遵循存储的行规则：仅追加，每行一个完整快照 `{ v: 1, at, id, record }`，每个 id 的最后一行是当前值，撕裂的最后一行被跳过，字段仅添加。设备不是存储记录：它们不携带 `seq`，从不进入事件日志、事件流或中继。

```ts
interface InboxDevice {
  id: string;              // `dev_<ULID>`（inboxId 获得 "dev" 前缀）
  name: string;            // 手机发送的名称；像线程名一样检查（checkInboxThreadName）
  platform: string;        // "ios"
  token_sha256: string;    // 整个令牌字符串的十六进制 SHA-256；令牌本身从不存储
  created_at: string;
  last_seen_at: string;    // 每个请求时保存在内存中；当其 UTC 日期变化时写入一行
  revoked_at: string | null;
  carriage: boolean;       // 中继承载此手机的项目和推送；配对时为 true，由手机的 relay 开关设置（9.2，第 4 节）
}
```

令牌是 `tok_` 加 32 个随机字节的 base64url（47 字符）。它仅由兑换返回一次。已撤销的记录保留其哈希，因此已撤销的令牌返回 `device_revoked`，而非 `device_token_invalid`。

每个设备还在计算机上保存其配对密钥，位于 `inbox/device-secrets/<dev id>`（模式 0600，目录 0700）：base64url 密钥，无其他内容。中继密钥从中派生（第 4 节）。撤销设备时删除该文件。

### 兑换 offer

`POST /api/inbox/device/pair`，唯一无令牌的入口路由。请求体：`{ secret, name, platform }`（QR 路径）或 `{ code, name, platform }`（手动输入数字，1.3）。应答：

- `201 { device, token, secret, computer: { name }, addresses: { tailnet, lan, fingerprint }, relay }`。`secret` 是 offer 的配对密钥，两条路径均返回，因此通过数字配对的手机也持有它。`relay` 在 Inbox 有邮箱时为 `{ url, mailbox_id }`，否则为 null。当邮箱存在时，Inbox 在应答前在中继注册新设备（第 4 节）；如果无法访问中继，则在其 socket 下次连接时注册。
- `410 offer_expired` 当没有匹配的开放 offer：从未创建、已使用、已过期、已替换、或因错误码关闭。
- `400 code_not_accepted_here` 在 LAN 监听器上，对无 `secret` 的 `code` 兑换：通过 Wi-Fi，手机仅通过 QR 配对（第 3 节）。窗口的端口和 tailnet 的监听器接受两者。offer 不受影响：不计错误尝试。
- `401 pairing_code_wrong { tries_left }` 错误码时。第五次错误码关闭 offer 并返回 `tries_left: 0`。
- `422 validation_error` 无 `secret` 或 `code`，或 `name` 无效。

首次兑换在尚无邮箱时创建邮箱（第 4 节）。

### 列出和撤销

- `GET /api/inbox/devices`（窗口路由）：`{ devices: [device without token_sha256] }`，未撤销的设备，最新在前。
- `POST /api/inbox/devices/:id/revoke`（窗口路由，窗口守卫）`{ serverSession? }`：`{ device }`。删除设备的密钥文件及其中继注册，并结束设备的开放事件流。未知 id 为 `404 device_not_found`。

路由应答的每个 `device`（7.2、7.25、7.36、7.37）是不含 `token_sha256` 的记录，包含 `carriage`；下文 7.2 和 7.25 为简洁起见在示例中省略 `carriage`。
- `POST /api/inbox/device/revoke`（入口路由，手机自己的令牌）：撤销调用设备，用于"移除此来源"（9.2），效果与窗口的撤销相同。`{ device }`。撤销两次不改变任何内容。

### 入口

每个手机请求都发往 `/api/inbox/device/*`。入口的检查，按顺序：

1. 任何 `Origin` 头：`403 origin_not_allowed`。浏览器永远不是设备。
2. `pair`：无令牌（见上）。
3. `Authorization: Bearer tok_...`。无：`401 device_token_missing`。无该哈希的设备：`401 device_token_invalid`。已撤销设备：`401 device_revoked`。
4. 路径在下方允许列表中，否则 `404 device_route_not_found`。
5. 允许列表中的 POST 在其请求体中携带 `idempotency_key`（第 6 节），否则 `422 validation_error`。两个例外是 `pair` 和 `revoke`，重复时无变化。两个原始请求体路由 `uploads` 和 `markups`（2026-10-15 添加）在查询中携带它（`?idempotency_key=`），因为其请求体是文件。
6. 请求在窗口的路径下交给窗口自己的处理器。入口不向路由、查询或请求体添加任何内容；手机不发送 `serverSession`，窗口的守卫接受（`checkServerSession`：缺失即通过）。手机的头部中仅传递 `Last-Event-ID`（事件流）、`If-None-Match`（图像路由的缓存验证器，2026-10-14 随 `messages/:id/image` 添加），因此缓存了图像的手机获得窗口的 `304`，以及在 GET 上的 `Range` 和 `If-Range`（播放器的字节范围，第 11 节）。在两个原始请求体路由上还传递 `Content-Type` 和 `Content-Length`，并原样流式传输请求体（含键）；手机放入该查询的 `serverSession` 由窗口像任何其他的一样检查，因此错误的为 `409 session_mismatch`。入口自行限制原始请求体，因为窗口的处理器在进程内运行，监听器的请求大小限制不适用：`uploads` 为 100 MB（`413 upload_too_large`），`markups` 为 32 MB（`413 markup_too_large`）。声明的 `Content-Length` 超过限制在读取任何字节前拒绝；无 `Content-Length` 的请求体（分块）在流式传输时计数并在限制处截断。两种应答均携带 `Connection: close`，且键下不记录任何内容。重放或拒绝后，入口仅读取并丢弃剩余请求体最多 1 MiB 或 2 秒，然后以 `Connection: close` 应答（客户端不应重用该连接）。"在该消息体中引用"规则限制代理消息暴露的内容，而非人员自己的客户端可读取的内容：已配对设备与窗口一样，可以发送引用项目根下任何 10 MB 或以下图像类型文件的回复，然后读取它，因此设备令牌与窗口一样强大（撤销丢失的手机），而通过中继，手机仅获得带内容类型的图像字节，因此应用必须将 SVG 作为不受信任的图像绘制。

入口应答之处。在回环监听器上，入口位于窗口旁。tailnet 到达第二个回环监听器，仅在"从我的 tailnet 访问"开启时开放，仅服务入口而不服务其他内容：`/api/inbox/device/` 之外的任何路径在那里返回 `404 device_route_not_found`，无论其 Host。`tailscale serve` 指向该监听器，绝不指向窗口的端口。在 LAN 监听器上仅存在入口（第 3 节）。

在 P1 中更正（来自 PR 1776 的审查）：初稿通过 Host 头将 tailnet 请求保留在入口上，假设"通过 `tailscale serve`，Host 是 MagicDNS 名称"。它是客户端自己的 Host：`tailscale serve` 将其传递（`r.Out.Host = r.In.Host`），因此发送 `Host: localhost` 的 tailnet 对等节点通过了回环允许列表并到达窗口的路由（配对 offer、`/mcp`、设置、重启）。现在分离来自 socket，如 LAN 监听器所做，且不为此读取任何 Host。

允许列表。每行是窗口当前的处理器，使用窗口使用的请求和应答（`docs/agent-guides/server-and-storage.md`，"Inbox Server"）。Exchange 编号指向第 7 节。

| 入口路由 | 窗口路由 | 请求 | 应答 | Exchange |
|---|---|---|---|---|
| `