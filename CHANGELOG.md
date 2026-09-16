# Changelog

## Unreleased — CLI 状态机重构（对齐老 CI）

### 设计哲学回归

完全对齐老分包 CI 的"状态机驱动"语义：CLI 不再提供"显式 upload / download / apply" 作为主要工作流，避免项目进入"半状态"。所有对外命令一条推进到稳定态。

### 对外命令（破坏性变更）

- **`init`** 重写为状态机驱动：复用 `PreChecker.inferState()`，一条命令推进到 Collecting（写配置 → 上传 → 预处理 → split(isProfile=true, referMd5) → 下载 → 应用）。幂等且支持断点续跑。首次必须 `-d <desc>`。
- **`split` → `dosplit`** 改名并扩展：内置 split + 下载 + 应用。默认 Profile；加 `--release` 即 Release。
- **新增 `getinfo`**：导出 `<project>/.plugincache/codesplit/gameinfo.txt`（兼容老 CI 脚本格式）。必须 Collecting 状态。支持 `-o <path>` 自定义路径。
- **新增 `collection-status`**：函数收集状态查询（独立于 `status`）。支持 `--watch`。
- **`disable` 新增别名 `restore`**。

### Hidden 命令（仅内部 debug）

`upload` / `download` / `apply` / `run` 改为 `hidden`，不在 `--help` 中展示，README 不再正式记录。仅供单步触发调试。

### 参数命名

- **bin 改名**：`wasm-split` → `wasmsplit-v2-ci`（包名 `@wasm-split/cli` 不变）。
- 全局 **`-w, --workspace` → `-p, --project-path`**（与老 CI 对齐）。
- `-o, --output-path` → `-o, --output`（getinfo / showfuncname）。
- 移除 `--opt-simd` / `--simdPatch`（当前后台未启用 SIMD patch）。

### 内部架构变更（shared）

- 新增 `CollectingFlowOrchestrator`：基于 `PreChecker.inferState()` 的状态循环。
- 新增 `SplitFlowOrchestrator`：dosplit 的 split + 下载 + 应用串联，前置校验状态合法性。
- 二者均加入 `createContext()` 的 `OrchestratorSet`。

### 迁移指南

```diff
- wasmsplit-v2-ci init -w /path -k key
- wasmsplit-v2-ci upload -w /path -k key -d "v1.0"
- wasmsplit-v2-ci split -w /path -k key --profile
- wasmsplit-v2-ci download -w /path -k key
- wasmsplit-v2-ci apply -w /path -k key
+ wasmsplit-v2-ci init -p /path -k key -d "v1.0"
+ wasmsplit-v2-ci getinfo -p /path -k key
+ wasmsplit-v2-ci dosplit -p /path -k key --release
```

配置目录：旧 `.wasm-split/` 不会自动迁移，升级后请重新 `init`。
