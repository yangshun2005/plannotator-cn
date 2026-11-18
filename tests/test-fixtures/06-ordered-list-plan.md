# 计划：切片 6 — API 补全 + 发布清理

## 背景

切片 1–5 已落地确定性冷启动、全部七种带 git 感知的热启动场景、具备单写者纪律的实时监听流水线、优雅关闭、周期性对账、SIGTERM/SIGINT 集成，以及同 cwd 咨询锁文件（以 `ConcurrentAccessError` 作为唯一公开错误类）。**226 个测试通过，剩余 10 个 `it.todo` 占位。** 这 10 个 todo 是库与规范之间仅剩的差距：

- `tree.search()` — `test/unit/query-search.test.ts` 中 3 个 todo；`src/query/list.ts:52` 的实现仍抛出 `NOT_IMPLEMENTED`。
- `tree.hash()` — `test/unit/query-hash.test.ts` 中 3 个 todo；`src/query/hash.ts:5` 的实现仍抛出 `NOT_IMPLEMENTED`。`src/filetree.ts:hash()` 也抛出。
- `reconciliation diff` — `test/unit/reconcile-diff.test.ts` 中 4 个单元 todo，覆盖新增/更新/删除分类及"不自动哈希"不变式。

外加两项从早期切片结转的覆盖率事项：

- 一个显式的空目录监听集成测试（切片 4 因依赖后端而推迟；切片 6 以按平台跳过的方式补上）。
- 一个通过 `vi.mock` 模拟 `src/watcher/parcel.js` 的冷订阅失败回归测试（切片 4 推迟；切片 5 的 `reconcile-warm-subscribe-failed.test.ts` 已确立该模式）。

以及一项发布门禁任务：既然切片 4 的实时监听 + 切片 5 的加固已证明跨运行时一致性，将 Bun CI 通道从**咨询性**翻转为**门禁**。

切片 6 落地后，`it.todo` 应归零，规范的每一节都有实际代码支撑，Bun 通道成为合并必需项。

## 锁定的设计决策（用户已签字）

### 1. `tree.hash()` — 感知新鲜度，更新时失效，删除时丢弃

契约是：**`tree.hash(path)` 返回文件当前磁盘内容的 SHA-256，前提是库的索引当前认为该路径是常规文件。** 结果缓存在 `files.hash` 中，每当 reconcile/watcher 更新该行时失效，每当该行被删除时丢弃。

### 2. `tree.search()` — AND 过滤语义，名称用 basename 子串不区分大小写，glob 作用于 cwd 相对路径

**查询形状**：`{ name?: string; glob?: string }`（已在 `src/types.ts:86` 中固定）。

## 验证

1. `npm run typecheck` 无报错。
2. `npm run lint` 无报错。
3. `npm test` — **预期约 251 个活跃通过**，**0 个 todo**，**0 个失败**。
4. `npm run build` 生成 `dist/` 及更新后的构建产物。
5. `npm run test:pack` 仍通过。
6. `npm run test:bun` 在 macOS 上通过，且**已移除 `continue-on-error`**。
7. **手动验证 — 哈希生命周期**：写一个文件，计算哈希，确认第二次调用命中缓存，修改文件，等待 watcher，重新计算哈希并确认摘要变化。
8. **手动验证 — 搜索语义**：仅 `name`、仅 `glob`、两者（AND）、空查询 → `[]`。
9. **手动验证 — 冷订阅失败**：确认拒绝后缓存目录已删除、锁文件已释放。
10. **手动验证 — Bun 门禁**：推送一个故意引入仅 Bun 失败的简单 PR，确认 CI 现在让 Bun 通道失败而非跳过。合并前还原。