# 模型提供商 OAuth

OpenBot 的 Google 和 xAI 模型连接与登录 OpenBot 或 CopilotKit 是分开的。两个提供商仍然可以使用 API 密钥。

## Google Gemini

Google OAuth 使用 Gemini Developer API 和所选 Google Cloud 项目的 API 配额。它不使用个人 Gemini 订阅。OAuth 请求使用 Google 的原生生成 API；本地服务器会转换现有代理的 Chat Completions 请求、流式回复、工具调用和截图。API 密钥连接继续使用 Google 的兼容 API。

在分发已配置的桌面构建之前，请在一个启用了 Generative Language API 的 Google Cloud 项目中注册一个 **Desktop app** OAuth 客户端。在构建桌面应用时设置以下变量以包含其默认值。运行时进程环境变量可以覆盖这些默认值：

- `OPENBOT_GOOGLE_MODEL_OAUTH_CLIENT_ID`：该桌面客户端的 ID。
- `OPENBOT_GOOGLE_MODEL_OAUTH_CLIENT_SECRET`：客户端密钥（如果已签发）。
- `OPENBOT_GOOGLE_MODEL_OAUTH_QUOTA_PROJECT`：提供 API 配额的项目。

对于 GitHub 构建的构建产物，请配置名为 `OPENBOT_GOOGLE_MODEL_OAUTH_CLIENT_ID` 和 `OPENBOT_GOOGLE_MODEL_OAUTH_QUOTA_PROJECT` 的仓库 Actions **变量**，以及当客户端有密钥时，配置名为 `OPENBOT_GOOGLE_MODEL_OAUTH_CLIENT_SECRET` 的 Actions **密钥**。Desktop 构建产物工作流和 Windows 签名工作流都会将这些设置传递给原生构建。它们会成为分发的桌面应用内部的默认值；桌面客户端凭据不是用户访问令牌或刷新令牌。

当以可复用工作流的方式调用 Desktop 工作流时，请显式传递其可选的 `OPENBOT_GOOGLE_MODEL_OAUTH_CLIENT_SECRET` 密钥，或使用 `secrets: inherit`。GitHub 不会向 Fork 的 Pull Request 提供仓库密钥。缺少设置不会导致构建失败；API 密钥连接仍然可用，而 Google 登录需要已配置的构建或运行时设置。

不要复用 OpenBot 的 Web SSO 凭据（`GOOGLE_OAUTH_CLIENT_ID` 和 `GOOGLE_OAUTH_CLIENT_SECRET`）：它们的回调和权限服务于不同的目的。在更广泛分发之前，请配置同意屏幕和测试用户，并完成 Google 的验证要求。

参考：[Gemini API OAuth](https://ai.google.dev/gemini-api/docs/oauth) 和 [Google desktop OAuth](https://developers.google.com/identity/protocols/oauth2/native-app)。

## xAI

xAI 登录使用设备授权：OpenBot 打开提供商的验证页面，用户批准显示的代码，然后 OpenBot 收到可刷新的凭据。API 调用使用 `https://api.x.ai/v1`。

该集成遵循 xAI 认可的 OpenCode 集成中的公共 OAuth 流程。分发者可以将 `OPENBOT_XAI_MODEL_OAUTH_CLIENT_ID` 设置为另一个已注册的客户端。账户权益和提供商限制仍然适用。

参考：[xAI 的 OpenCode 集成](https://x.ai/news/grok-opencode) 和 [xAI OAuth 发现](https://auth.x.ai/.well-known/openid-configuration)。

## 其他构建时设置

`VITE_OPENBOT_SUPPORT_URL` 替换启动失败时提供的支持链接。未设置时，它指向此仓库的 Issue。白标构建将其指向自己的支持渠道。

## 凭据生命周期

桌面的提供商凭据保留在其本地部署中；React 不会收到访问令牌或刷新令牌。本地服务器在模型调用之前刷新凭据，并保存轮换后的令牌。代理进程收到的是本地代理凭据，而不是提供商的刷新令牌。提供商撤销需要重新登录。

不要提交部署凭据文件，也不要将其内容包含在支持报告中。OAuth 验证必须包括一次真实的模型调用、刷新和重启，以及一个使用其浏览器工具并渲染图形组件的 Bot。