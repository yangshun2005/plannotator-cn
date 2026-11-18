# 验证：收件箱各标签页共享一个事件连接

变更内容：同一浏览器源下的每个收件箱标签页共享一个事件流（`packages/inbox/sharedEvents.ts`）。Web Locks 领导者持有 EventSource，并通过 BroadcastChannel 转发每一帧；跟随者只监听。参见 docs/agent-guides/inbox-notifications-and-agents.md 中的“每个浏览器一个事件流”。

所有运行：macOS，从本分支构建的编译二进制（`.local/plannotator`），临时数据目录位于操作系统临时目录下（绝不用 `/tmp`，绝不用 `~/.plannotator`），`PLANNOTATOR_INBOX_CONNECT=0`，`NO_BROWSER`，无头浏览器，通知完全打桩（无操作系统通知），声音只计数（从不播放）。“流”指服务器自身统计的打开事件流数量（`/api/inbox/health` 上的 `event_streams`），减去测试的参考流。

如何检查标签页：每个标签页记录其处理的记录（`window.__plannotatorInboxEventLog`）。基准是测试从开始就持有的参考流。“精确”指标签页在其订阅游标之后恰好处理了参考流的各行，每行一次，按顺序。

## 浏览器验证（Playwright，同一浏览器上下文中的真实标签页）

| 用例 | 浏览器 | 结果 |
| --- | --- | --- |
| 1 个标签页 | Chromium | 1 个流，精确，notify_reach 窗口为 1 |
| 3 个标签页 | Chromium、WebKit | 1 个流，每个标签页精确，窗口为 3 |
| 8 个标签页 | Chromium、WebKit | 1 个流，每个标签页精确，窗口为 8 |
| 12 个标签页 | Chromium | 1 个流，每个标签页精确，窗口为 12；每个标签页加载 440-495 ms |
| 旧构建（origin/main），第 7 个标签页 | Chromium | 6 个标签页加载（约 460 ms）；第 7 个在 8 s 内未加载；关闭一个标签页后它在 534 ms 内加载：六连接限制 |
| 新构建，8 个打开时第 9 个标签页 | Chromium / WebKit | 在 467 ms / 3 s 内加载，1 个流 |
| 领导者关闭时有 agent 正在写入（约 48 条消息） | Chromium、WebKit | 另一个标签页在 1 ms 后成为领导者（Chromium）；1 s 内（WebKit）；每个标签页精确；1 个流 |
| 领导者重载时有 agent 正在写入 | Chromium、WebKit | 另一个在 8 ms 内成为领导者；每个标签页（重载的那个从其新游标开始）精确；1 个流 |
| 领导者崩溃（`Page.crash`，无 unload）时有 agent 正在写入 | Chromium | 另一个在 2 ms 内成为领导者；每个标签页精确；1 个流 |
| 领导者冻结（`freeze`，主线程被占用，`resume`）时有 agent 正在写入 | Chromium | 另一个在 217 ms 内成为领导者；冻结的标签页在恢复后追上；每个标签页精确；1 个流 |
| 领导者停滞（调试器占用其主线程 8 s，无 freeze 事件）时有 agent 正在写入，一个跟随者在前 | Chromium | 前面的标签页在 3.4 s 时取得锁（`steal`）；其最长无记录间隔 3.2 s；最多 2 个流（77 个采样）；旧领导者在 8.0 s 时（再次运行时）得知失去锁，关闭其流并跟随（9 ms 后）；每个标签页精确（4/4）；之后 1 个流 |
| 领导者被附加 HTML 页面中的循环脚本停滞（沙箱化 srcdoc 框架，8 s），一个跟随者在前 | WebKit | 前面的标签页在 3.8 s 时成为领导者；最长间隔 3.7 s；最多 2 个流；旧领导者在 8.1 s 时得知并跟随；每个标签页精确（3/3） |
| 上述每个用例，前面的标签页 | Chromium | 最长无记录间隔：关闭 48 ms，重载 47 ms，崩溃 46 ms，冻结 235 ms |
| 通知，4 个标签页全在后台 | Chromium | 总共 1 条通知和 1 次响铃 |
| 一个跟随者在前，领导者在后台 | Chromium | 1 条通知，由前面的标签页发出；0 次响铃 |
| 前面标签页打开的线程中有一个问题 | Chromium | 0 条通知，0 次响铃 |
| notify_reach：跟随者关闭 / 崩溃 / 领导者关闭 / 标签页打开 | Chromium | 4→3 用 1 ms，3→2 用 27 ms（崩溃：领导者看到其锁被释放），2→1 用 1 ms，保持在 1 达 17 s（超过宽限期），1→2 用 26 ms |
| notify_reach：跟随者关闭，领导者关闭 | WebKit | 每一步都在超时内（见 results-webkit.json） |
| 收件箱重启（停止，在同一端口启动） | Chromium | 启动后 1.2 s 再次为 1 个流；窗口为 4；每个标签页精确 |
| 两个源（`localhost` 和 `inbox-second.localhost`，代表 tailnet 地址） | Chromium | 2 个流，每个源一个领导者，每个标签页精确 |
| 无 Web Locks（由初始化脚本移除） | Chromium、WebKit | 每个标签页各自一个流（3 个标签页 3 个流），与之前相同 |
| 浸泡：5 分钟，标签页随机打开、关闭、重载和崩溃（87 次打开，51 次关闭，48 次重载，34 次崩溃），同时 agent 写入 5,577 条消息 | Chromium | 231 个采样中流从未超过 1；每个存活的标签页精确；每个离开的标签页按顺序且从不重复 |
| 浸泡，最终构建上的第二次运行：4 分钟（68 次打开，40 次关闭，35 次重载，27 次崩溃），4,501 条消息 | Chromium | 179 个采样中流从未超过 1；4/4 存活标签页精确；所有 102 个离开的标签页按顺序且从不重复 |
| 浸泡，审查修复后：4.5 分钟（41 次打开，24 次关闭，51 次重载，19 次崩溃，17 个标签页被带到前台，7 个领导者被调试器停滞 3-9 s，全部 7 个被接管），4,996 条消息 | Chromium | 停滞之外 176 个采样中流从未超过 1；停滞期间 179 个采样中从未超过 2；存活的标签页精确；所有 94 个离开的标签页按顺序且从不重复 |
| 从另一站点 `GET /api/inbox/events` | 服务器 | `Sec-Fetch-Site` 跨站 / 同站，外部 Origin 和 `Origin: null`：403 `cross_origin`，不计数；同源、`none`、无元数据（手机的门、agents）和页面自身的 Origin：200 |
| 页面错误 | 两者 | 无 |

审查修复后的最终运行，在合并 origin/main 的 head 上重复（Chromium 14 个测试用 6.4 分钟含 4.5 分钟浸泡，WebKit 7 个用 37 s，两次全部通过；数字来自第一次，第二次在几 ms 内：停滞 3.38 s / 间隔 3.17 s，WebKit 停滞 3.78 s / 3.71 s，浸泡 5,050 条消息，7 个停滞全部被接管，停滞之外流从未超过 1，停滞期间从未超过 2）：接管 1 ms（关闭）、8 ms（重载）、2 ms（崩溃）、223 ms（冻结）、3.4 s（停滞）、12 ms（WebKit 关闭）、3.8 s（WebKit 停滞）；旧构建（origin/main）使第 7 个标签页停滞 8 s，并在一个标签页关闭后 533 ms 加载它，新构建在 459 ms 内加载第 9 个；收件箱重启在 1.2 s 内重连；notify_reach 步骤 0/27/0/28 ms。

## 第二次审查

- 一个标签页在其追赶运行期间成为领导者，追赶随后失败，过去会永久持有其自身流的每一帧（审查者的探针：3 s 后 5 条记录中 0 条）。它现在清除其持有的内容并处理其流。单元测试 `a tab that comes to lead while its catch-up runs…` 在之前的代码上失败，现在通过；带停止领导者的随机压力测试现在也会使追赶失败（一些在慢回答的末尾），在之前的代码上也失败。
- CI（GitHub 的 Linux 运行器）：崩溃的领导者被接管用了 2.6 s，而界限是 1 s（macOS 上 2-3 ms）。崩溃标签页的锁在浏览器注意到渲染器死亡时释放，这在 Linux 上约需 3 s：下一次 CI 运行记录接管为 `queue`（锁按序到来，而非通过 steal）在崩溃后 3.0 s，崩溃的跟随者在 3.3 s 后停止计数（该运行器上的其他用例：关闭 3 ms，重载 23 ms，冻结 567 ms，停滞 3.6 s 通过 `steal`，WebKit 停滞 5.1 s，每个标签页精确）。崩溃用例的界限现在是停滞领导者的接管（7 s：每 3 s 检查，2 s 内无回答，为慢运行器留余地），允许短暂出现两个流，每次接管记录其方式（`queue` 或 `steal`；本地除停滞外每个用例都是 `queue`）。冻结界限是 2 s，reach 证明的跟随者崩溃步骤是 8 s。
- `inbox-shared-windows.test.ts` 的“after the whole browser goes…”等待计数，而非通知（过期计时器在宽限期后 1 ms 触发）：它现在等待通知，20 次运行 20 次通过。
- 在再次合并 origin/main 的 head 上重跑：单元 26 通过，服务器 `packages/server/inbox*` 689 通过（694 个测试），DOM 564 通过，shared-events e2e Chromium 12 通过（旧构建验证和浸泡跳过：未设置基线二进制或浸泡时间）和 WebKit 7 通过。

## 单元和服务器测试

| 测试 | 结果 |
| --- | --- |
| `packages/inbox/sharedEvents.test.ts`（Web Locks 带 `steal`、BroadcastChannel、服务器的流、以及主线程不运行任何东西的暂停标签页的伪造）：1/3/12 个标签页，游标跳过，served_by，所有帧类型，写入期间领导者关闭/崩溃，跟随者消失（崩溃和关闭），冻结/恢复，前进/后退缓存，新领导者后面的睡眠跟随者，迟到的订阅者，失败的追赶在下一次 hello 时重试，停止的领导者被可见跟随者接管并在再次运行时跟随，从不回答的领导者不被接管，隐藏跟随者按隐藏节奏，失败的追赶保留其间隙（审查者的 [3,4] 探针现在给出 [1,2,3,4]），无订阅跟随时早期通道关闭，垃圾消息，取消订阅，40 个种子随机生命最多 12 个标签页，30 个种子随机生命含领导者停止和标签页显示与隐藏（最多 2 个流，每条记录一次） | 25 通过；4 个新审查测试在之前的代码上失败 |
| `packages/server/inbox-shared-windows.test.ts`：served_by，gone，通过交接的宽限期，两个互相命名的标签页从不计数，再远一个链接从不计数，整个浏览器消失（跟随者在领导者的流之后一个宽限期停止计数，页面得知），event_streams，once=1，跨站拒绝 | 7 通过 |
| `packages/server/inbox*.test.ts`（tailnet，含门） | 685 通过，0 失败（690 个测试，在合并的 head 上） |
| `packages/inbox` DOM 测试（`DOM_TESTS=1 --isolate`），运行回退路径（happy-dom 无 Web Locks） | 562 通过（合并的 head） |
| 类型检查（`bun run typecheck`） | 干净 |
| 读取事件流的现有 e2e 规范（Chromium，编译二进制）：`notifications`、`inbox-sound`、`inbox`、`inbox-reply-drafts`、`inbox-send-now`、`inbox-open-stays-put`、`inbox-seen-on-sight`、`inbox-restart`、`inbox-updates`，以及 main 的新 `inbox-picks-composing`、`inbox-refresh` | 74 通过，1 跳过（与 main 相同） |

## 未验证

- 真实的 Chrome 冻结（节能模式）。无头页面始终可见，所以 `Page.setWebLifecycleState` 无效；验证播放相同的序列（`freeze`，主线程被占用，`resume`）。
- 从不再次运行的领导者（无限循环）：其他标签页立即接管（它们不需要它运行），但其标签页在服务器上保持其流打开，直到浏览器杀死它；期间 2 个流。
- 所有标签页隐藏时，隐藏节奏（15 s）是浏览器允许的：Chrome 将长期隐藏标签页的计时器节流到每分钟一次，Safari 暂停后台标签页；仅在伪造中测量，未在浏览器中。
- iOS Safari 自身对后台领导者的暂停（由审查者推断）：由相同检查覆盖，未在设备上运行。
- Firefox，以及 Safari 本身（仅通过 Playwright 的 WebKit）。
- tailnet 地址本身（另一个源，由第二个回环名称代表）。
- 同一源上旧版和新版收件箱构建的混合标签页（旧标签页保持其自己的流；未运行）。