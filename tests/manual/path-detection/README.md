# 路径检测手动测试

测试代码文件路径检测、智能解析、存在性校验、baseDir 传递，以及歧义匹配选择器。

## 快速开始

```bash
cd tests/manual/path-detection
chmod +x *.sh

# 计划模式（仓库内主要场景）
./run-plan.sh --keep

# 标注模式（仓库内主要场景）
./run-annotate-in-tree.sh --keep

# 标注模式（仓库外主要场景 — 关键的 baseDir 测试）
./run-annotate-out-of-tree.sh --keep
```

`--keep` 阻止退出时清理沙箱，便于检查文件。不加则脚本退出时删除临时目录。

每个脚本都会：构建 Hook、创建含模拟仓库和外部夹具的临时沙箱、启动对应服务器、打开浏览器。

## 沙箱内容

```
$SANDBOX/
├── repo/                               ← 模拟项目根目录（已 git init）
│   ├── packages/editor/App.tsx         ← 歧义基名（1/2）
│   ├── packages/review-editor/App.tsx  ← 歧义基名（2/2）
│   ├── packages/ui/components/Button.tsx ← 唯一基名
│   ├── packages/ui/index.ts
│   ├── src/utils/helper.ts             ← 缩写路径目标
│   ├── src/config.json                 ← 非 .ts 代码文件
│   ├── app/[slug]/page.tsx             ← Next.js 方括号路由
│   ├── node_modules/junk/App.tsx       ← 必须被忽略
│   └── test-plan.md                    ← 夹具计划
└── external/                           ← 仓库外
    ├── notes.md                        ← annotate-out-of-tree 主要文件
    ├── script.ts, config.yaml, bar.ts, parent.ts
    ├── design.md                       ← 链接文档目标
    └── sub/subdoc.md, sub/nested.ts    ← 嵌套链接文档
```

## 检查清单

### 计划模式（`run-plan.sh`）

| § | 用例 | 预期 |
|---|------|----------|
| 1A | `packages/editor/App.tsx`（反引号，完整） | 链接 → 打开文件 |
| 1B | `packages/ui/components/Button.tsx`（正文，完整） | 链接 → 打开文件 |
| 1C | `src/config.json`（反引号，JSON 扩展名） | 链接 → 打开文件 |
| 2A | `editor/App.tsx`（缩写） | 链接 → 打开 packages/editor/App.tsx |
| 2B | `utils/helper.ts`（缩写，正文） | 链接 → 打开 src/utils/helper.ts |
| 2C | `./editor/App.tsx`（前导 ./） | 链接 → 打开 packages/editor/App.tsx |
| 3A | `Button.tsx`（唯一基名） | 链接 → 打开 packages/ui/components/Button.tsx |
| 3B | `App.tsx`（歧义基名） | 带徽标的链接 → 选择器 → 两个条目 |
| 3C | `helper.ts`（唯一基名） | 链接 → 打开 src/utils/helper.ts |
| 4A | `packages/ui/shortcuts/core.ts`（不存在，反引号） | 纯 `<code>`，不可点击 |
| 4B | `packages/ui/shortcuts/runtime.ts`（不存在，正文） | 纯文本，非链接 |
| 5A | `packages/ui/{core,runtime}.ts`（花括号） | 纯 `<code>`，非链接 |
| 5B | `packages/ui/*.tsx`（glob） | 纯 `<code>`，非链接 |
| 5C | `some path/with spaces/file.ts`（含空格） | 纯 `<code>`，非链接 |
| 6A | `app/[slug]/page.tsx`（方括号路由） | 链接 → 打开文件 |
| 6B | `[slug]/page.tsx`（缩写方括号） | 链接 → 打开 app/[slug]/page.tsx |
| 7A | `junk/App.tsx`（node_modules） | 纯文本，或仅匹配真实 App.tsx |
| 8A | 带 .ts 扩展名的 URL | 仅 URL 链接，无路径泄漏 |
| 8B | 同一行 URL + 真实路径 | 两个独立链接 |
| 8C | Wikipedia 风格括号 URL | 单个 URL 链接 |
| 9 | 围栏代码块内的路径 | 块内无链接 |
| 10 | HTML 注释内的路径 | 不可见，无链接 |
| 11A | `../script.ts`（计划中无 baseDir） | 纯文本（降级） |
| 12 | 点击链接文档 → external/notes.md | 浮层打开；验证内部路径 |

### 标注仓库外（`run-annotate-out-of-tree.sh`）

| 用例 | 预期 |
|------|----------|
| notes.md `script.ts` | 链接 → 打开 external/script.ts |
| notes.md `config.yaml` | 链接 → 打开 external/config.yaml |
| notes.md `editor/App.tsx`（跨上下文） | 链接 → 通过 cwd 向上查找打开仓库的 packages/editor/App.tsx |
| notes.md `packages/ui/shortcuts/core.ts` | 纯文本（各处均不存在） |
| 点击 [Open design doc] | 浮层；`bar.ts` → external/bar.ts |
| 点击 [Open subdoc] → subdoc.md | 浮层；`../parent.ts` → external/parent.ts |
| subdoc.md `nested.ts` | 链接 → 打开 external/sub/nested.ts |
| subdoc.md `../missing.ts` | 纯文本（不存在） |

### 网络面板检查

- 计划/标注加载时：恰好 **一次** `POST /api/doc/exists`
- 链接文档浮层打开时：**额外一次** POST
- 计划模式 POST 体：`{ paths: [...] }`（无 `base` 字段）
- 标注仓库外 POST 体：`{ paths: [...], base: "<external dir>" }`
- 链接文档 POST 体：`{ paths: [...], base: "<linked doc's parent>" }`