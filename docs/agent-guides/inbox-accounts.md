## 收件箱：账户与用量

设置 → 账户（`#settings/accounts`）显示本机及所有已链接机器上的 Claude 账户，以及每个账户的用量。这是已批准的“账户页”工作（`.product/approved/accounts-page/`，不在本仓库）：步骤 1 和 2 是读取，步骤 3 是在本机切换账户（见下文，已构建但关闭），页面是已批准重设计的安静版本：每台机器一个区块，每行是一个登录的名称、一个进度条及其数字，其余信息悬停显示。侧边栏的用量行、其预览以及手机上的面板显示同样的读取结果，且从不切换。在已链接机器上切换（步骤 4）、Codex、OpenCode 和 Pi 是后续步骤，各自单独评审。真正的切换要等一次测试：两个真实登录，且所有者在机器旁；在那之前，只针对假的 `cswap` 验证过。

**切换已关闭**（`packages/core/inbox-types.ts` 中的 `INBOX_ACCOUNT_SWITCHING`，值为 `false`；所有者，2026-11-06：“现在就发布页面，切换保持关闭直到测试”）。它是一个常量，由服务器和窗口读取，不是设置项。关闭期间，每台机器的 `canSwitch` 都为 false，因此每一行都是普通 `div`（没有按钮、没有焦点、没有“Switch”字样、没有悬停着色、没有确认），桌面和手机皆如此；未使用的行不显示空心圆（`data-blank`，所有者的选择：空心圆看起来可点击；其列保留，使名称对齐，使用中标记保留）；服务器没有 `POST /api/inbox/accounts/switch`：该路径落到 `404 not_found`，不执行任何操作。切换代码、其测试及以下规则保持不变：切换的测试仅在常量开启时运行（`skipIf`），关闭状态的测试仅在关闭时运行。开启切换就是改那一行。

收件箱通过 claude-swap（`cswap`，MIT，由用户安装）读取。Plannotator 从不读取、存储或移动凭据。`cswap list --json` 可能会：为获取用量，它可能刷新过期的登录，消耗一次性令牌并写回新令牌，这一切在 claude-swap 内部完成，如同输入 `cswap list`。没有跳过此行为的标志。没有开关设置：`cswap` 在收件箱的 PATH 上时页面显示账户，不在时页面说明如何安装。查找从不超出该 PATH，因为查找 home 目录可能让测试触达真实的 `cswap`，所以收件箱启动后才安装的 `cswap` 需要重启，页面会说明这一点。

**读取**（`packages/server/inbox-accounts.ts`）。`GET /api/inbox/accounts` 用 `AgentEnvironment.which` 查找 `cswap`，并通过 `runUntilExit` 运行 `cswap list --json`（无 shell，argv 数组）。**绝不因运行慢而杀死它。** 在令牌已消耗、新令牌尚未写入之间杀死会丢失登录，且 `list` 没有信号处理器，所以 SIGTERM 也不更安全。20 秒时调用方收到 `timed_out`，运行继续。只有运行到 2 分钟仍在进行才连同其进程组一起杀死，视为真正卡住。收件箱停止会让运行完成，之后没有东西执行该限制。不要改成 `runWithTimeout`，那是给 agent CLI 用的。答案为以下之一：

- `{ state: "not_installed" }`：不在 PATH 上，或运行退出码 127，即 shell 的“命令未找到”（工具已消失的 shim）。
- `{ state: "failed", reason }`：`timed_out`、`run_failed`（非零退出）、`bad_output`（不是列表），或 `unknown_schema`（本构建不读取的 `schemaVersion`；窗口提示更新）。
- `{ state: "accounts", accounts }`。

读取的规则：

- **仅具名字段。** 每个值都在 `account()` 和 `usageWindow()` 中按名称选取：`number`、`alias`、`email`、`organizationName`、`isOrganization`、`active`、`usageStatus`、`usageError`、`usageRetryAt`、`usageAgeSeconds`，以及 `usage` 的 `fiveHour`、`sevenDay` 和 `scoped`（各含 `pct`、`resetsAt`、`countdown`、`clock`；scoped 窗口还含 `name`）。绝不传递原始 JSON 或展开行：后续 `cswap` 新增的字段绝不能到达窗口。未知额外字段被忽略。
- **不出现 `cswap` 的字样。** 其 stdout、stderr 和错误文本从不记录、存储或返回。失败是固定代码。`usageStatus` 和 `usageError` 仅作为单个词传递，窗口将其映射为自己的文本。
- **API key 行没有地址。** `cswap` 给它一个占位符（`api-key-3@token.local`）。服务器丢弃它，针对状态 `api_key` 及任何 `@token.local` 地址。
- **`cswap` 未报告的窗口为 null，绝不为 0。** 非有限数字的 `pct` 丢弃其窗口，没有剩余窗口的 `usage` 读作无读数。0% 且无重置时间的窗口按报告显示：mock 隐藏它，该差异在存在真实 Ultra 读数前保持决定。超过 `resetsAt` 的窗口读作未报告：`cswap` 在获取失败时提供其最后良好读数，所以数字来自该重置之前。
- **一次只运行一个，直到真正结束。** 运行期间的调用方加入它并等待自己的 20 秒，所以 `timed_out` 后的刷新仍可得到该运行的答案，且绝不启动第二次运行。新的收件箱进程不知道上一个进程留下的运行，所以两者可能重叠；claude-swap 自己的锁覆盖了这一点。不写磁盘，不轮询：运行在窗口、页面或预览打开时，或刷新时启动。
- **每个收件箱每 30 秒最多一个进程，无论谁请求。** 每次运行的答案（失败的也是）在内存中保留 `INBOX_ACCOUNTS_CACHE_MS`，所以重新加载的手机每次加载不启动进程，损坏的 `cswap` 无论窗口打开多少次只运行一次。刷新发送 `fresh=1`，来自本机窗口或已链接机器，仅在距上次运行启动已过 30 秒后才重新读取；该时间内得到最后读数。保留时间超前于当前时间说明时钟被调回，视为已过，所以读取从不等待时钟追上。`cswap` 消失立即显示，因为发现这一点不启动进程。这节省进程，不节省登录：`cswap` 无论运行多频繁，最多每 180 秒（`SERVE_TTL_S`）获取并刷新一次，登录安全依赖该下限。在读取之间改变答案的测试传 `cacheMs: 0`；针对编译二进制的 spec 在其沙箱中设 `PLANNOTATOR_TEST_ACCOUNTS_CACHE_MS=0`（`accountsTestOptions`）。

谁可读取：窗口的读取守卫（同源，`403 cross_origin`）。所有者像任何窗口路由一样通过 tailnet 读取，手机浏览器即如此读取。门仅从已链接机器的令牌接受 `accounts`，所以没有手机应用和没有中继命令能到达它。收件箱仅支持 Bun，所以 Pi 服务器中没有。

**已链接机器的读取**（`packages/server/inbox-machines.ts` 中的 `accountsOf`）。`GET /api/inbox/accounts/<machine>` 通过其 `PeerLink` 询问该机器的门，如同机器间每次读取，远端收件箱运行自己的读取，有自己的运行、30 秒答案和 `fresh`。它在上述唯一守卫之后，`accounts` 被通用转发拒绝，对所有人：转发的读取没有所有者检查，按原样传递远端答案。规则：

- **仅在请求时。** 远端读取在本地读取发生时发生：窗口、页面或预览打开，或刷新。不轮询，机器恢复不是读取。本地不保留任何东西，内存或磁盘皆无：GET 答案不存储，所以地址绝不到达任一侧磁盘。
- **再次仅具名字段。** 远端答案由 `parsePeerAccounts` 用与 `cswap` 相同的 `account()` 选取，因为另一端可能更旧、更新或恶意。其失败保持固定代码。超过 1,024 字符的字段被丢弃，超过前 200 的行被丢弃，超过 `INBOX_MACHINE_ACCOUNTS_MAX_BYTES`（1 MiB，是一次运行可打印量的 16 倍）的正文不再读取并返回 `failed`、`bad_output`，无论其状态。`PeerLink.request` 还通过 `readCapped` 以 64 KiB 查看 401 以识别撤销的链接：此路径上对远端正文的读取没有无上限的。
- **一次一台机器。** 窗口在各自请求中询问每台机器并跟踪各自的读取，所以慢机器只拖住自己的行，刷新时亦然：发现某机器正在读取的刷新在该读取结束时再次询问。
- **等待为 25 秒（`INBOX_MACHINE_ACCOUNTS_WAIT_MS`），超过远端运行的 20 秒，** 且涵盖正文。所以慢的远端运行返回为其自己的 `timed_out`，其措辞来自 claude-swap，运行继续服务下一次读取。`peer.request` 以 `timeoutMs` 获取它：链接通常的 15 秒会先停止。
- **`{ state: "offline" }` 是完全不回答的机器：** 线路断开（立即，无请求），或读取失败且线路也断开。绝不是错误。
- **`{ state: "unreachable" }` 是机器在线但对此读取未给答案：** 401、403 或 500，非 JSON 正文，门在整个等待期间挂起但线路仍在线，或其撤销的链接。等待超过线路的 20 秒静默限制，所以真正休眠的机器届时已失去线路。本收件箱不回答窗口亦然。该行显示“收件箱未回答。”
- **`{ state: "needs_update" }` 是版本差异：** 较旧的门返回 `404 device_route_not_found`，本构建无法读取的对象（更新的 `state`，一行无法选取）也来自较旧的门。链接、其线程和写入继续。太旧无法链接的机器亦然。
- **门仅服务自己收件箱的账户。** 它只保留查询的 `fresh`，且 `/api/inbox/accounts/<machine>` 不是门路由，所以没有机器能传递第三方的。

**窗口**（`packages/inbox/components/Accounts.tsx`，类 `ib-acct-*`；记录设计 `.product/approved/accounts-page/mock/` 和 `approval.md`，不在本仓库）。该部分在设置中机器之后，刷新在其标题旁。页面每台机器一个区块：机器图标和名称在细线之上，本机在前，然后每个已链接机器按 `/api/inbox/machines` 给出的顺序。区块内每个工具一个列表：目前仅 Claude Code，带其标记（`CLAUDE_CODE`，一个 `AccountsTool`：主机、名称、交换工具、其安装和添加命令）。Codex 列表作为同一机器下的第二个 `AccountsTool` 加入；页面不假设只有一个工具。

每台机器的列表是自己的，如其自己的 `cswap` 打印：跨机器不合并，慢机器只拖住自己的区块（其位置显示“正在读取账户……”）。没有列表的区块在行应在处用几个词说明（`Quiet`，`[data-machine-state]`）：“休眠”（mock 的用词，设置 → 机器对 `offline` 的用词）、“更新 Plannotator”、“收件箱未回答。”或读取的失败行。没有 `cswap` 的机器得到未安装区块，与本机相同。慢机器的答案不移动其区块上方任何东西；其行落地时下方区块下移，如 mock 所示。

列表（`ladder`）是使用中的账户在前，然后其他：有读数的，然后无读数的，然后 API key，各按剩余空间最多（最紧数字最低者在前），然后 `cswap` 的编号。一行是一个点（使用中的账户填充，最紧限制达 90% 或以上时琥珀色，切换到它失败后红色）、名称、弱化注释和用量簇。所有者要求安静的行：名称、一个条及其数字，无标签文本。簇是两固定列的网格，沿页面对齐，每台机器的行在一列：最接近耗尽的限制的条（`tightest`：最高舍入数字，平局取第一个）及其数字。不显示限制名称；它作为 `data-usage` 和 `aria-label` 附在数字上。其余在行的悬停 `title`（`rowDetail`）中，每行一个事实：每个限制（“5h 21% · Fable 33% · week 58%”）、最近限制的重置时间、旧读数的“最后读取于 <age> 前”，以及“最多空间”。数字显示已用量，如 `cswap` 报告，未使用行上弱化。90% 或以上琥珀色，按舍入数字判断，使数字与颜色从不矛盾。API key 仅其名称，无条；“API key，按 token 计费”是其悬停标题。`usage: null` 也无条，原因在注释中（`noReadingReason`，知道 `cswap` 的状态）。

名称旁的注释，用“ · ”连接：该列表两行读取相同时的组织；行无读数的原因，或需要人（“重新登录”等）；以及另一机器列表有同一登录在使用时的“在 <machine> 上”（`usedElsewhere`，按地址和组织；API key 无地址，所以绝不在另一机器上）。“在 <machine> 上”是共享限制警告，仅在此处说明。“最多空间”（`mostRoom`）仅在一行的悬停标题中，且仅当使用中的账户有可信读数达 90% 或以上：有登录的可信行（非 API key，不需要人）中紧数字最低者，若低于使用中的。可信读数是 10 分钟内且非 `relogin_required`。

注释的空间（`RowNotes`，`.ib-acct-notes`）：长名称先截断，注释占行文本最多 45%。其中组织或原因（`.ib-acct-cut`）先缩短，“在 <machine> 上”（`.ib-acct-on`）有自己的框，保留其词。手机上注释在名称下方换行，具名 API key 在此加“API key”，因手机无悬停。每行的悬停词也在行内视觉隐藏的 `.ib-sr` span（`[data-account-detail]`）中，因普通行上的 `title` 屏幕阅读器无法到达。

行读作别名，否则地址开头，否则“API key”。个人 Claude 组织按地址命名，`isOrganization` 对它们也为 true。所以窗口各处，包含行自身地址的组织不算组织（`accountOrganization`）：它会把地址放到页面上。同一列表仍读相同的行在名称后安静显示 `cswap` 的编号，`mira #2`，仅这些行。该编号是机器自己的，切换会送回。页面任何地方不显示地址。

超过 10 分钟的读数（`usageAgeSeconds`；`cswap` 正常服务年龄为 3 分钟）意味着 `cswap` 无法刷新，因获取失败时它提供最后良好读数。其数字弱化且绝不琥珀色，使用中账户的详情说“最后读取于 2h 前”。

所有者对此页的规则：“只显示我需要的，直到我想操作。想操作就带我去设置页。不要叙述性标签或冗余信息。”所以它无导语无旁注，只有账户及其数字，其他状态都是几个词。保持如此。

未安装显示“**claude-swap** 未安装在 <machine> 上。”、claude-swap README 的第一个安装命令在复制块中，以及“已安装？在那里重启收件箱。”无保存账户显示“登录 Claude Code，然后保存账户”及 `cswap add`。失败显示短行。在良好列表上失败的刷新保留列表并在其上方显示该行；首次读取失败仅显示该行。首次读取显示等待行，刷新保持列表并禁用按钮旋转直到本机回答。

**切换**（步骤 3；`packages/server/inbox-accounts.ts` 中的 `switchTo`，`POST /api/inbox/accounts/switch`，`Accounts.tsx` 中的 `AccountRow` 和 `SwitchConfirm`）。这一切在 `INBOX_ACCOUNT_SWITCHING` 开启时。本机每个未使用的行就是切换：普通按钮（`button[data-account]`，`aria-expanded`）。桌面上悬停着色并在右侧显示安静的“Switch”；悬停样式在 `@media (hover: hover)` 下，所以手机点击不留着色，手机行无“Switch”字样且高 44 px 或以上。按行在其下打开确认（`[data-accounts-confirm]`，焦点在取消），再按或取消关闭。它是一短行“切换到 <name>？”，然后取消和切换：所在区块命名机器和工具，所以仅其 `aria-label` 说“在 <machine> 上将 Claude Code 切换到 <name>？”，行的“在 <machine> 上”已说明登录共享。API key 的确认加弱化“按 token 计费”（`[data-accounts-billed]`）：关乎金钱，手机无悬停。打开的会话在切换后说一次：切换后的读取显示登录在使用时，该行下显示弱化“重启打开的会话以使用它”（`[data-accounts-restart]`），6 秒后淡出（`RESTART_NOTE_MS`）。失败或未确认的切换绝不显示。确认持有登录（地址、组织、别名），非 `cswap` 的编号：终端中 `cswap swap` 后的读取将其保持在该登录下新编号，无该登录的读取或现已使用的读取关闭它。手机上确认按钮在问题旁，高 40 px。切换