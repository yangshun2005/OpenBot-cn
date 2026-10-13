# Windows 桌面签名

构建使用根 OpenBot 发布版本号加上 `-internal.g<commit>`。工作流会验证打包的应用和安装程序都嵌入了该版本号，并将 `build-version.json` 与二进制文件一起包含。参见[桌面构建版本](releasing.md#desktop-build-versions)。

[桌面 Windows 签名工作流](../.github/workflows/desktop-signing.yml)使用 Azure Key Vault 中现有的 DigiCert 证书构建 OpenBot 及其 NSIS 安装程序。它将已验证的二进制文件和签名证据保留为 Actions 构建产物，保留 14 天。它不会创建或发布发布版本。

普通的桌面 CI 和 Fork PR 构建保持未签名状态。`tauri.windows-signing.conf.json` 覆盖配置仅由受保护的签名作业显式传递给 Tauri。不要将其重命名为 `tauri.windows.conf.json`：Tauri 会自动将该文件名合并到每个 Windows 构建中。

## 请求签名验证构建

要对 Pull Request 的构建进行签名，请向同一仓库的 PR 添加 `windows-signing` 标签，并批准其 `windows-signing` 环境部署。工作流会从打标签事件中检出确切的 PR head SHA。新的推送会运行回归作业；请移除并重新应用标签以对新 SHA 进行签名。Fork PR 无法进入此签名作业。没有 `pull_request_target` 触发器。

要对任意 ref 进行签名，请使用 **Actions → Desktop Windows signing → Run workflow**，选择要验证的 ref，并将 `signing-mode` 设置为 `keyvault`。默认的 `none` 仅运行无需凭据的回归测试。环境审查者在批准访问发布者密钥之前，应检查确切的源 SHA 和工作流变更。

成功的签名运行会使用 7-Zip 从 NSIS 安装程序中提取 `openbot-desktop.exe`，然后使用 Windows Authenticode 和 `signtool verify /pa /all /v /tw` 验证**该载荷**以及单个 `*-setup.exe` 安装程序。签名必须有效、带有时间戳，且发布者为 `Tawkit, Inc.`。任何警告或 SignTool 非零退出码都会导致作业失败。`signatures.json` 记录源 SHA、构建产物 SHA-256 哈希、签名者和时间戳证书；配套的文本文件保留详细的 SignTool 输出。只有在两者都通过后才会上传二进制文件。提取的应用从 `desktop/signed-app/` 保留：Tauri 在打包后会恢复未签名的构建可执行文件，因此验证 `target/release/openbot-desktop.exe` 会检查错误的副本。这些检查不会测试 SmartScreen 信誉，也不会运行应用 UI。

## 一次性基础设施设置

使用受保护的 GitHub 环境 `windows-signing` 并配置必需的审查者。配置以下环境**变量**；它们包含公共标识符，而非密码：

| 变量 | 值 |
| --- | --- |
| `AZURE_CLIENT_ID` | `cb923310-e793-4557-929e-b33e49a42297` |
| `AZURE_TENANT_ID` | `c3050389-57ad-4c62-8dcd-fe5e2af4fbce` |
| `AZURE_SUBSCRIPTION_ID` | 包含 `cpk-signing-kv` 的订阅 |
| `AZURE_KEY_VAULT_URL` | `https://cpk-signing-kv.vault.azure.net` |
| `CODE_SIGNING_CERT_NAME` | `code-signing` |

现有 Entra 应用程序的所有者，或具有适当授权的应用程序管理员，必须添加联合凭据。从仓库根目录执行：

```sh
az ad app federated-credential create \
  --id cb923310-e793-4557-929e-b33e49a42297 \
  --parameters desktop/signing/azure-federation.json
```

请先检查现有凭据；不要重复或替换其他仓库的凭据。检入的 JSON 中的主体是 OpenBot 已验证的不可变主体，包含所有者和仓库 ID，作用域限定于此环境。`Insufficient privileges` 响应需要授权的应用所有者/管理员来运行该命令；GitHub 环境批准不会授予 Entra 权限。

现有签名身份需要证书读取和密钥签名权限。使用 Key Vault RBAC 时，**Key Vault Certificate User** 加上 **Key Vault Crypto User** 即可覆盖这些操作；仅 Crypto User 不授予证书读取访问权限。在已配置的情况下复用现有证书和权限。不需要客户端密钥、导出的私钥、PFX 或 Tauri 更新程序签名密钥。

工作流固定使用 Azure Login 和 AzureSignTool 7.0.1，检查下载工具的 SHA-256，并通过 GitHub OIDC 获取 Key Vault 访问令牌。包装器会为每次签名调用刷新令牌，将其注册用于日志掩码，并在之后清除其进程环境。令牌绝不会写入工作流输出、`GITHUB_ENV` 或构建产物。Tauri 会为应用和安装程序以及需要签名的 NSIS 组件调用该包装器。

## 检查脚本

```sh
pwsh -NoProfile -File desktop/scripts/test-windows-signing.ps1
```

回归测试套件使用合成命令结果来测试缺失配置、原生失败、未签名/被篡改的签名、发布者不匹配、缺失时间戳、缺失/过期的安装程序，以及两个文件的证据。它在没有 Azure 访问权限的 Windows PR CI 上运行。只有受保护的签名运行才能证明证书、OIDC 联合和真实 Windows 签名链协同工作。

参考：[Tauri 自定义签名](https://v2.tauri.app/distribute/sign/windows/)、[AzureSignTool 7.0.1](https://github.com/vcsjones/AzureSignTool/tree/v7.0.1)、[Windows SignTool 验证](https://learn.microsoft.com/en-us/windows/win32/seccrypto/signtool)、[GitHub OIDC 主体](https://docs.github.com/en/actions/reference/security/oidc)、[Entra 联合凭据](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust)。