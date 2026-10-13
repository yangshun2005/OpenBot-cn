# macOS 麦克风验证

`src-tauri/Info.plist` 提供了麦克风权限描述。Tauri 会将其与生成的元数据合并，包括 CI 使用的 `build-version.plist` 覆盖层。`bundle.macOS.entitlements` 指向 `Entitlements.plist`，当应用使用强化运行时（hardened runtime）签名时，该文件启用音频输入。

从 `desktop/` 目录构建并检查一个临时签名的应用：

```sh
bun install --frozen-lockfile
bun test scripts/desktop-version.test.ts
bun scripts/desktop-version.ts internal
APPLE_SIGNING_IDENTITY=- bun run tauri build --config src-tauri/tauri.build-version.conf.json --bundles app
plutil -extract NSMicrophoneUsageDescription raw src-tauri/target/release/bundle/macos/OpenBot.app/Contents/Info.plist
codesign --verify --deep --strict src-tauri/target/release/bundle/macos/OpenBot.app
codesign -d --entitlements - src-tauri/target/release/bundle/macos/OpenBot.app
codesign -dv src-tauri/target/release/bundle/macos/OpenBot.app
```

描述必须非空，签名的权限 `com.apple.security.device.audio-input` 必须为 `true`，且签名标志必须包含 `runtime`。配置回归测试在现有的桌面版本测试套件中运行。这些检查验证的是打包，而非实时麦克风采集。

如需进行实时检查，请使用一台麦克风正常且已配置语音提供商的 Mac：

1. 启动打包后的 `OpenBot.app`，完成设置，并在桌面窗口中打开其工作区。测试浏览器标签页或 `tauri dev` 无法验证打包应用的权限。
2. 开始听写并批准 OpenBot 的 macOS 麦克风提示。说一段简短的话，停止，然后验证转录结果。如果权限已被拒绝，请在系统设置 → 隐私与安全性 → 麦克风中启用 OpenBot，然后重新打开应用。
3. 启动并停止一次 OpenAI 语音对话，然后进行一次 Grok 语音对话。验证每次都能接收语音输入，并在停止后释放麦克风。
4. 在麦克风权限被拒绝的情况下重复上述操作，验证 UI 报告失败，而不是停留在录音状态。

记录测试的应用构建版本、macOS 版本、提示结果以及所有三条采集路径的结果。临时构建无法验证 Developer ID 签名或公证；在发布前，请在已签名的分发构建产物上重复测试。

参考：[Tauri 原生配置与权限](https://v2.tauri.app/distribute/macos-application-bundle/#native-configuration)、[WebKit 的用途描述要求](https://bugs.webkit.org/show_bug.cgi?id=217104)。