# 桌面遥测

遥测默认启用。Welcome 会披露该功能，且无需选择加入。使用 `COPILOTKIT_TELEMETRY_DISABLED=true` 或 `DO_NOT_TRACK=1` 启动 OpenBot 可同时禁用桌面发射器和运行时。这两个变量均接受 `true` 或 `1`。更改环境变量后请退出并重新启动。选择退出会移除待处理的桌面事件，并阻止记录或重放。

原生发射器在凭据可用之前就会启动。它会在应用的本地数据目录中持久化一个随机的安装 UUID 和一个有界队列。它不会从机器派生身份标识。运行时通过 `CPK_TELEMETRY_ID` 接收相同的 UUID，并使用采样率 1；选择退出仍然优先。

## 事件允许列表

所有桌面事件名称均以 `oss.desktop.` 开头。Rust 的 `EventData` 枚举和 `schema_json()` 定义了封闭的属性 Schema；反序列化会拒绝未知属性和枚举值。

| 事件后缀 | 属性 |
| --- | --- |
| `step_viewed` | 设置步骤枚举 |
| `harness_chosen` | Harness 枚举，包括 `byo_url` |
| `model_chosen` | 提供商和凭据路径枚举；自定义基础 URL 布尔值 |
| `engine_detected` | 引擎枚举；响应布尔值 |
| `engine_installed` | 引擎和安装程序结果枚举 |
| `windows_stage` | Windows 先决条件结果枚举 |
| `image_pull` | 结果枚举、毫秒数、可选的观测下载字节数 |
| `setup_failed` | 设置步骤和错误类别枚举 |
| `activated` | 无属性；首次成功设置 Bot 回答 |
| `setup_abandoned` | 最后设置步骤枚举 |

每个事件都携带桌面发行版、数字应用版本、平台、架构、可用的数字操作系统版本以及引擎。运行时通过其现有的 `telemetryProperties` Hook 接收相同的桌面元数据。桌面 Schema 不接受任何提示词、回答、凭据、文件名、路径、主机名、电子邮件、模型名称、YAML 值或自定义 URL。

镜像计时涵盖仅缺失时的 Compose 拉取，不包括启动和迁移。字节计数是从 Compose 观测到的下载计数器，按层去重；它们不是精确的网络总量。缺失的计数器保持为 null。不支持 `pull --policy missing` 的旧版 Compose 提供程序保留其现有的隐式拉取行为，并且不发出拉取测量数据。

## 投递与验证

事件在发送前持久化，并在重试期间保持其事件 ID。队列最多容纳 256 个事件，溢出时丢弃最旧的事件。后台投递、有界退出刷新以及重新启动时的重放均使用现有的 CopilotKit 摄取契约。激活前的崩溃会在下次启动时记录为放弃。遥测失败不会阻止设置。

摄取端必须接受新的 `oss.desktop.*` 命名空间：[oss-path-to-production #290](https://github.com/CopilotKit/oss-path-to-production/pull/290)。仅 HTTP 响应成功并不能证明下游已接受；生产摄取也会确认被过滤的事件。该更改的合并和部署是生产桌面投递的必要条件。

回归测试位于原生遥测和拉取指标模块、前端遥测测试以及服务器元数据测试中。要进行实际的本地 HTTP 和已安装运行时检查，请构建原生探针并从仓库根目录运行驱动程序：

```sh
cargo build --manifest-path desktop/src-tauri/Cargo.toml --release --example telemetry_probe
bun desktop/scripts/validate-telemetry.ts desktop/src-tauri/target/release/examples/telemetry_probe
```

该驱动程序使用临时数据和回环接收器。它演练了独立进程的持久化/重放、退出、激活去重、选择退出以及已安装运行时的身份/元数据交接。它不需要 AI 凭据，也不会向生产环境发送验证事件。Windows 和 Linux 的执行仍由平台 CI 运行覆盖；Mac 运行并不能证明其原生 UI 行为。

本地 HTTP 验证可通过 `desktop/scripts/validate-telemetry.ts` 与单独构建的 `telemetry_probe` 示例复用。它检查具有稳定安装/事件 ID 的离线队列重放、恢复的放弃、退出刷新、跨进程重启仅激活一次、原生/运行时元数据，以及 `COPILOTKIT_TELEMETRY_DISABLED` 和 `DO_NOT_TRACK` 的 `true`/`1` 两种值。选择退出必须移除排队状态和身份标识，并产生零次 HTTP 发送。

9 月 12 日的本地运行使用了已安装运行时 1.70.1 及其公开的 ESM Hono 工厂，遵循生产环境的 `eventsource` 预加载。其现有的 Bun 依赖布局需要显式设置 `NODE_PATH` 指向现有的提升目录，以便缓存的 `gaxios` 模块能够解析 `extend`。未安装或修改任何依赖。从仓库根目录复现该适配器：

```sh
NODE_PATH="$PWD/node_modules/.bun/node_modules" bun --no-env-file --no-install desktop/scripts/validate-telemetry.ts /absolute/path/to/telemetry_probe
```

最终的回环报告记录了五个原生请求和一个运行时请求，且安装身份匹配，所有四个选择退出用例均通过。这覆盖了发射器和已安装运行时的 HTTP 契约；全新依赖安装和最终的 PostHog 投递仍需要各自的验证。