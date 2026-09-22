# 变更日志

本文件记录 **Wasm代码分包V2** 插件与 **wasmsplit-v2-ci** CLI 的变更。两者同版本号、同内容。
所有条目按 [Keep a Changelog](https://keepachangelog.com/zh-CN/) 格式维护。

> 本文件由发布流程在发版时自动写入，请勿手工修改（见 `docs/tech/build-release.md`）。

## [2.1.5] - 2026-09-22

### Added

- 支持多包融合场景：子包名与路径改为从 wx-game-kit 配置派生，融合后（代码树位于 minigame / minigame-iOS 等目录）可正常分包；单包项目行为不变。

### Changed

- （IDE）「包体优化」分组更名为「微信优化项」，与文档口径一致。
- （IDE）版本信息补全 MD5、描述与时间；收集页增加全平台收集提醒。

### Fixed

- 修正构建产物中中文被转义为 \uXXXX，导致模板与 CLI 产物不可读的问题。
- 日志时间改为本地时区。

## [2.1.4] - 2026-09-20

### Changed

- （CLI）初始化与分包过程的输出面向使用者重写：新增工程基本信息（AppID、API 版本、WXGameKit 版本）；上传阶段显示文件名与大小；预处理显示等待时长与完成后的函数总量。

### Fixed

- （CLI）发布包内补齐变更日志文件。

## [2.1.3] - 2026-09-16

### Added

- （CLI）新增 `switch-md5` 命令，用于切换工程代码分包 md5。

### Changed

- （IDE）插件名称统一为「Wasm代码分包V2」，详情页补齐功能说明、使用流程与常见问题。

### Fixed

- （IDE）修正插件内显示的版本号不正确的问题。

## [2.1.2]

首个 V2 对外版本。V2 为通用适配方案的分包插件与命令行工具，与 V1 为独立版本线，版本号互不相干；支持 Unity 以及 Unity 之外的引擎（UE、Cocos、自研引擎等）。

### Added

- （CLI）`init`：一条命令把工程推进到函数收集态（上传 → 预处理 → 分包 → 下载 → 应用），支持断点续跑。
- （CLI）`dosplit`：执行分包并自动下载、应用产物（默认 Profile，加 `--release` 为 Release）。
- （CLI）`getinfo`：导出函数收集信息（gameinfo.txt）。
- （CLI）`showfuncname` / `reportfuncname`：导出与上报函数名列表。
- （CLI）`config`：读写工程配置。
- （CLI）`disable`（别名 `restore`）：还原工程到未分包状态。
- （IDE）提供侧边栏可视化操作界面，覆盖从初始化到产物应用的全流程。
- 增量分包：指定历史版本复用已有的函数收集结果（依赖导出时生成的 symbol 文件）。

### Note

- 与 V1 分包插件不能混用：通用适配方案（`game.json` 的 `plugins` 为 `WXGameKit`）用 V2；Unity 专用方案（为 `UnityPlugin`）用 V1。

> V1（Unity 专用，`master` 分支）的变更日志见 [WasmSplit.md](https://github.com/wechat-miniprogram/minigame-unity-webgl-transform/blob/main/Design/WasmSplit.md#changelog)。
