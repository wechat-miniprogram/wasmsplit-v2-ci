# @wasm-split/cli

WASM 代码分包管理工具 - CLI 命令行工具（状态机驱动）

## 设计理念

CLI 完全对齐老分包 CI 的**状态机驱动**语义：
**一条命令推进项目到稳定态**，避免显式的 upload/download/apply 让项目陷入"半状态"。

- `init`：写配置 + 上传 + 预处理 + 分包 + 下载 + 应用 → 推进到 **Collecting**
- `dosplit`：分包 + 下载 + 应用 → 拿到可用产物
- `getinfo`：导出函数收集信息（gameinfo.txt）

底层用 `PreChecker.inferState()` 与 VSCode 扩展共用同一套状态判定，CLI 与 UI 互通。

## 安装

```bash
npm install -g wasmsplit-v2-ci
```

## 快速开始（CI 三行）

```bash
# 1. 推进到收集态（首次必填 -d）
wasmsplit-v2-ci init -p /path/to/project -k /path/to/private.key -d "v1.0.0"

# 2. 导出函数收集信息（输出 .plugincache/codesplit/gameinfo.txt）
wasmsplit-v2-ci getinfo -p /path/to/project -k /path/to/private.key

# 3. 做 Release 分包并自动应用产物
wasmsplit-v2-ci dosplit -p /path/to/project -k /path/to/private.key --release
```

## 参数约定（对齐老 CI）

| 参数                            | 说明                                   |
| ------------------------------- | -------------------------------------- |
| `-p, --project-path <path>`     | 项目根目录绝对路径                     |
| `-k, --private-key-path <path>` | 私钥文件路径                           |
| `-d, --desc <description>`      | 版本描述（init 首次必填）              |
| `-o, --output <path>`           | 输出文件路径                           |
| `--release`                     | Release 分包（dosplit 默认 Profile）   |
| `--refer-md5 <md5>`             | 依赖的历史版本 MD5                     |
| `--watch -i <ms>`               | 监听模式（status / collection-status） |

## 命令列表

### `init` — 状态机推进到 Collecting

一条命令把项目推到收集态。内部按 `PreChecker.inferState()` 循环调度：
`NotInitialized` → 写配置/扫描/激活 → `ReadyToUpload` → `Upload` →
`Preprocessing` → 轮询 → `ReadyToSplit` → `Split(isProfile=true, referMd5)` →
`ReadyToDownload` → 下载 → `ReadyToApply` → 应用 → `Collecting`（退出）。

```bash
wasmsplit-v2-ci init -p <project> -k <key> -d "v1.0.0" [--refer-md5 <md5>] [--no-scan]
```

- 幂等：可重复执行；已完成的步骤会自动跳过。
- 首次执行必须 `-d <desc>`；后续重跑可省略。

### `dosplit` — 分包 + 下载 + 应用

前置：项目已处于可分包态（建议先 `init`）。完成 split → 自动轮询 → 自动下载 → 自动应用。

```bash
wasmsplit-v2-ci dosplit -p <project> -k <key> [--release] [--refer-md5 <md5>]
                   [--opt-profile] [--opt-func]
```

- 默认 Profile 分包，加 `--release` 即 Release。
- `--opt-profile` / `--logCallInWasm`：Profile 性能优化（互为别名）。
- `--opt-func` / `--mergeSmallFunc`：函数量优化（互为别名）。

### `getinfo` — 导出 gameinfo.txt

前置：必须处于 Collecting 状态（否则报错）。

```bash
wasmsplit-v2-ci getinfo -p <project> -k <key> [-o <path>]
```

- 默认输出：`<project>/.plugincache/codesplit/gameinfo.txt`
- 内容：`JSON.stringify(ProfileInfo)`（兼容老 CI 的脚本读取格式）
- stdout 同步打印关键字段（profileIncrementFuncNum / sourceFuncNum / profileFuncNum 等）

### `status` — 查询完整状态

```bash
wasmsplit-v2-ci status -p <project> -k <key> [--watch -i 10000]
```

任何状态可用，含版本/分包/收集摘要。

### `collection-status` — 查询函数收集状态

```bash
wasmsplit-v2-ci collection-status -p <project> -k <key> [--watch -i 10000]
```

类似 `status`，但只展示收集相关字段。

### `disable` / `restore` — 还原项目

```bash
wasmsplit-v2-ci disable -p <project>
# 或
wasmsplit-v2-ci restore -p <project>
```

### `showfuncname` — 导出函数名

```bash
wasmsplit-v2-ci showfuncname -p <project> -k <key> [-o <path>]
```

### `reportfuncname` — 上报函数名

```bash
wasmsplit-v2-ci reportfuncname -p <project> -k <key> -r <funcname.txt>
```

### `config` — 配置 CRUD

```bash
wasmsplit-v2-ci config get  <key>   -p <project>
wasmsplit-v2-ci config set  <key> <value> -p <project>
wasmsplit-v2-ci config list -p <project>
```

### `version` — 版本号

```bash
wasmsplit-v2-ci version [--verbose]
```

## Hidden 命令（仅内部 debug）

以下 4 个命令保留为隐藏入口，不在 `--help` 中展示，**不建议在 CI 中使用**：单独执行会让项目处于"半状态"。

- `upload`：仅上传，不推进后续
- `download`：仅下载，不应用
- `apply`：下载+应用，不触发新分包
- `run`：FullFlow（基于 step tracker；与 init 等价但不基于状态推断）

调试时可使用 `wasmsplit-v2-ci upload -p ... -k ... -d "debug"` 类似形式。

## 配置目录

CLI 与 VSCode 扩展共用配置目录：

```
<project>/.plugincache/codesplit/
├── config.json       # 配置文件
├── gameinfo.txt      # getinfo 输出（默认）
├── logs/             # 日志
└── tmp/              # apply 的中转目录
```

## CI/CD 集成

### GitHub Actions

```yaml
name: WASM Split
on:
  push: { branches: [main] }

jobs:
  split:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }

      - run: npm install -g wasmsplit-v2-ci

      - name: Init project (state-machine driven)
        run: |
          wasmsplit-v2-ci init \
            -p ${{ github.workspace }} \
            -k ${{ secrets.PRIVATE_KEY_PATH }} \
            -d "CI build ${{ github.sha }}"

      - name: Export gameinfo
        run: |
          wasmsplit-v2-ci getinfo \
            -p ${{ github.workspace }} \
            -k ${{ secrets.PRIVATE_KEY_PATH }}

      - name: Release split
        run: |
          wasmsplit-v2-ci dosplit \
            -p ${{ github.workspace }} \
            -k ${{ secrets.PRIVATE_KEY_PATH }} \
            --release

      - uses: actions/upload-artifact@v4
        with:
          name: split-files
          path: |
            wasmcode/*.wasm.br
            wasmcode1/*.wasm.br
            .plugincache/codesplit/gameinfo.txt
```

## 破坏性变更（v2.0 → 当前版本）

- **参数命名**：`-w, --workspace` 全部改为 `-p, --project-path`（与老 CI 对齐）。
- **配置目录**：`.wasm-split/` → `.plugincache/codesplit/`（与 VSCode 扩展共用 `PLUGIN_CACHE_DIR`）。升级后需重新 `init`。
- **命令重塑**：`split` 改名 `dosplit` 且内置 download+apply；新增 `getinfo` / `collection-status`；`upload` / `download` / `apply` / `run` 改为 hidden。
- **删除**：`--opt-simd` / `--simdPatch` 选项（当前后台未启用 SIMD patch）。

## 架构

CLI 通过 `@wasm-split/shared` 的 `ContextProvider` 接线 Service / Orchestrator，与 VSCode 扩展共享 95%+ 业务代码：

- **PreChecker**：状态推断（与 UI 共用）
- **CollectingFlowOrchestrator**：init 的状态机循环
- **SplitFlowOrchestrator**：dosplit 的 split+下载+应用串联
- **CLIAuthAdapter / CLIUIAdapter**：仅平台特定的鉴权与日志输出

## License

MIT
