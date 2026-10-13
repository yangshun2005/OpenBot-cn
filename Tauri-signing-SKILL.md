---
name: agui-tauri-windows-signing
description: >
  使用 AG-UI 组织存储在 Azure Key Vault 中的 DigiCert 代码签名证书
  （不可导出的 RSA-HSM 密钥），通过 GitHub OIDC 认证的 AzureSignTool
  进行签名，为 Tauri Windows 构建（.exe / .msi）配置并排查 Authenticode
  代码签名问题。涵盖 tauri.conf.json 的 signCommand 接线、CI 签名接缝
  （模式门控：none | keyvault）、Authenticode 签名、Tauri 更新器
  （minisign）密钥与 NuGet 包作者签名之间的区别，以及 Azure/Entra 访问
  模型——包括当 Tauri 应用位于与现有 NuGet 签名通道不同的仓库时会发生
  哪些变化。适用场景：为 Tauri 应用添加 Windows 签名、将发布工作流接线
  到密钥保管库（同仓库或跨仓库）、调试签名、登录、权限或 SmartScreen
  故障，或回答"哪个证书 / 哪个密钥 / 哪个工具 / 哪个访问权限签什么"。
---

# 使用 Azure Key Vault 进行 Tauri Windows 签名

## 背景：什么签什么

组织的签名基础设施有**三种独立机制**。切勿将它们混为一谈——它们使用
不同的密钥、不同的工具和不同的验证路径：

| 机制 | 密钥 | 工具 | 适用范围 |
|---|---|---|---|
| NuGet **作者签名** | DigiCert 证书，CN=Tawkit, Inc. — 私钥位于 Azure Key Vault（不可导出，RSA-HSM） | `NuGetKeyVaultSignTool` | 仅 `*.nupkg` |
| **Authenticode**（本技能） | 同一 DigiCert 证书，同一密钥保管库 | `AzureSignTool` | Windows PE 二进制文件：`.exe`、`.dll`、`.msi`（包括 Tauri 构建） |
| 程序集强命名 | `AGUI.snk` | MSBuild | .NET 程序集；与上述两种签名无关 |
| Tauri **更新器**签名 | Tauri minisign 密钥对（不是密钥保管库证书） | `tauri signer` | `tauri-plugin-updater` 的更新清单 |

代理经常搞错的额外事实：

- nuget.org 在推送时会自动应用其自己的**仓库签名**。作者签名才是让包在
  库中变为"已签名"状态的关键；Authenticode 完全不需要它。
- 符号包（`*.snupkg`）**有意不**进行作者签名。
- 密钥保管库证书：DigiCert Trusted G4 Code Signing RSA4096 SHA384
  2021 CA1 颁发者链，OV 验证。由于是 OV（而非 EV），Windows
  SmartScreen 信誉会随着已签名版本按发布者逐步积累，而不是立即授予——
  全新应用出现看似未签名的警告是预期行为，直到信誉积累起来，这不是签名
  失败。
- 私钥**永远不会离开 HSM**。每一次签名——无论是 NuGet 还是
  Authenticode——都是在密钥保管库内部执行的 `keys/sign` 操作。
  切勿尝试导出密钥或生成 `.pfx`。

NuGet 通道的参考实现（本技能所镜像的模式）：
`.github/workflows/publish-release.yml`（签名步骤）和
`sdks/dotnet/docs/signing.md`（设计文档）。

## 任务：将签名接入 Tauri 构建

### 1. 将 `signCommand` 指向包装脚本

在 `tauri.conf.json` 中，覆盖默认的 `signtool.exe` 调用。
Tauri 会为每个二进制文件**以及**每个安装程序各调用一次此命令，
用每个文件路径替换 `%1`：

```json
{
  "bundle": {
    "windows": {
      "signCommand": "pwsh ./scripts/sign-tauri.ps1 %1"
    }
  }
}
```

不要配置旧版的 `certificateThumbprint` / `digestAlgorithm` /
`timestampUrl` 字段——这些字段期望本地安装的证书和本地可访问的密钥，
无法访问 Key Vault HSM。

### 2. 创建包装脚本

`scripts/sign-tauri.ps1`——镜像 `publish-release.yml` 中的
NuGetKeyVaultSignTool 步骤（相同的密钥保管库 URL、证书名称、令牌、
摘要和时间戳权威）：

```powershell
param([string]$File)
azuresigntool sign `
  -fd sha256 `
  -tr http://timestamp.digicert.com `
  -td sha256 `
  -kvu $env:AZURE_KEY_VAULT_URL `
  -kvc $env:CODE_SIGNING_CERT_NAME `
  -kva $env:AZURE_ACCESS_TOKEN `
  $File
if ($LASTEXITCODE -ne 0) { throw "Signing failed for $File" }
```

来自 NuGet 通道的标志映射（确认你保持了它们一致）：

| NuGetKeyVaultSignTool | AzureSignTool |
|---|---|
| `--file-digest sha256` | `-fd sha256` |
| `--timestamp-rfc3161 http://timestamp.digicert.com` | `-tr http://timestamp.digicert.com` |
| `--timestamp-digest sha256` | `-td sha256` |
| `--azure-key-vault-url` | `-kvu` |
| `--azure-key-vault-certificate` | `-kvc` |
| `--azure-key-vault-accesstoken` | `-kva` |

### 3. 使用 OIDC 接线工作流（无需长期密钥）

发布任务使用 GitHub OIDC 联合身份认证到 Azure——与 NuGet 通道相同的
模式。不要引入客户端密钥。

```yaml
- uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_SIGNING_CLIENT_ID }}   # federated to GH OIDC
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    allow-no-subscriptions: true
- run: dotnet tool install --global AzureSignTool
- run: |
    $env:AZURE_ACCESS_TOKEN = (az account get-access-token `
      --resource https://vault.azure.net --query accessToken -o tsv)
    npm run tauri build
```

工作流任务必须声明 `permissions: id-token: write`（以及
`contents: read`）——否则 GitHub 根本不会颁发 OIDC 令牌。
这是新工作流或仓库中首次运行登录失败最常见的原因。

### 4. 保留模式门控（安全默认值）

遵循 `publish-release.yml` 中既定的 `signing-mode` 接缝：
工作流输入默认为 `none`（未签名，在任何密钥保管库配置之前可安全合并），
只有 `keyvault` 才启用签名。对于 Tauri，通过以下任一方式实现门控：

- 仅在发布任务中合并包含 `signCommand` 的
  `tauri.windows.conf.json` 覆盖文件，或
- 从基础配置中省略 `signCommand`，当 mode = keyvault 时作为显式的
  构建后步骤对构建出的 `.exe` / `.msi` 进行签名。

两者均可接受；不变式是未配置的环境永远不会因为接缝的存在而失败
（或静默签名）。

## 访问权限：什么会变，什么不会变

密钥保管库侧的权限模型对于 NuGet 签名和 Authenticode 是**完全相同**的。
AzureSignTool 恰好需要：**Key: Sign**、**Secret: Get**、
**Certificate: Get**——与 NuGetKeyVaultSignTool 在每次发布时已经执行的
三个操作相同。因此：

- **不需要新的 Azure RBAC 角色分配**（现有的 Key Vault Crypto User，
  或旧版访问策略，已经覆盖了这些操作）。
- **不需要更改密钥保管库配置**——同一密钥保管库、同一证书、同一
  `keys/sign` 操作。
- **不需要新的访问策略，不需要导出密钥，不需要生成 `.pfx`。**

唯一可能需要 Entra ID 更改的是将 GitHub OIDC 令牌映射到应用注册的
**联合凭据**，且仅当 Tauri 构建从当前凭据未覆盖的位置运行时才需要。

### 同仓库 / 同环境 → 无需更改

如果 Tauri 签名任务在与现有 NuGet 签名任务相同的环境或 ref 下于
`ag-ui-protocol/ag-ui` 中运行，现有的联合凭据即可覆盖。现有的
`AZURE_SIGNING_CLIENT_ID` 和 `AZURE_TENANT_ID` 直接可用。

### 不同仓库 → 添加一个联合凭据

联合凭据主体是**精确匹配**的——没有通配符，没有组织范围的
`repo:org/*` 覆盖。作用域为 `repo:ag-ui-protocol/ag-ui:...` 的凭据
不覆盖任何其他仓库。在 NuGet 通道使用的**同一**应用注册上，添加一个
新的联合凭据：

- **颁发者：** `https://token.actions.githubusercontent.com`
- **受众：** `api://AzureADTokenExchange`
- **主体：** 新仓库的令牌主体，例如
  `repo:other-org/tauri-app:environment:release` 或
  `repo:other-org/tauri-app:ref:refs/heads/main`

主体格式注意事项（对 2026 年及以后很重要）：2026 年 7 月 15 日之后
创建的仓库默认发出**不可变主体格式**——
`repo:owner@owner-id/repo@repo-id:ref:...`，内嵌数字 ID。
如果 Tauri 仓库是全新的，其 OIDC 令牌的 `sub` 声明使用该格式，
凭据主体必须与之精确匹配。在创建凭据之前，运行一个调试工作流，
请求 OIDC 令牌并打印 `sub` 声明；逐字复制它。

多仓库的替代方案：Entra **灵活联合**（声明匹配表达式）允许单个凭据
覆盖多个仓库——当第三个或第四个签名仓库变得可能时，优先使用它。

### 跨仓库检查清单（完整）

在**另一个**仓库中：

1. 在现有应用注册上添加联合凭据（见上文）。
2. 设置仓库密钥/变量：`AZURE_TENANT_ID`、`AZURE_SIGNING_CLIENT_ID`
   （与 ag-ui 仓库相同的值），以及 `AZURE_KEY_VAULT_URL` 和
   `CODE_SIGNING_CERT_NAME`。密钥**不会**跨仓库共享——要么按仓库
   重新创建，要么将非敏感值（租户 ID、客户端 ID、密钥保管库 URL、
   证书名称）提升为组织级 Actions 变量，让两个仓库读取同一数据源。
3. 如果凭据主体是环境作用域的，创建签名环境（例如 `release`），
   并对其应用保护规则。
4. 在签名任务上声明 `permissions: id-token: write`。
5. 在任务中安装 AzureSignTool：`dotnet tool install --global AzureSignTool`。
6. 添加 `tauri.conf.json` 的 signCommand + 包装脚本（仓库本地）。

### 影响范围：共享身份 vs 独立身份

将凭据添加到现有应用注册意味着 Tauri 仓库共享了能够对组织的代码签名
密钥调用 `keys/sign` 的完全相同身份：任何能在该仓库中触发签名任务的
人都可以用 Tawkit 证书签署任何东西。由于证书是 OV，SmartScreen 信誉
会累积到共享发布者——任何仓库中签名任务被攻陷都会损害组织整体的签名
信誉。在 Azure 中零成本的缓解措施：

- 将联合凭据作用域限定为**受保护环境**（主体
  `...:environment:release`）并设置必需审核者，而不是普通分支 ref，
  这样签名仅在批准后运行。
- 在新仓库中保留 `signing-mode: none | keyvault` 门控，与 ag-ui 接缝
  一致，这样未配置的 PR 永远不会触发签名。

如果反而需要隔离，为 Tauri 仓库创建一个**单独**的应用注册——这是
唯一**确实**需要新 Azure 访问权限的变体，因为新的服务主体必须手动
授予密钥保管库上的 Key Vault Crypto User。

## 验证

构建后，在发布前验证：

```powershell
# Authenticode + certificate chain + RFC 3161 timestamp
signtool verify /pa /v /all path\to\App.exe

# PowerShell alternative
Get-AuthenticodeSignature path\to\App.exe | Format-List

# Expected: Status Valid, SignerCertificate subject CN=Tawkit, Inc.,
# TimeStamperCertificate from DigiCert, timestamped within the cert validity.
```

对于 NuGet 包（独立通道，同一密钥保管库）：
`dotnet nuget verify Package.nupkg --verbosity detailed`。

让新仓库上线，按顺序：

1. 调试工作流：请求 OIDC 令牌并打印 `sub` 声明。
2. 使用该精确主体创建联合凭据。
3. `azure/login`，然后只读冒烟测试：
   `az keyvault certificate show --vault-name <vault> --name <cert>`。
4. 如果冒烟测试通过，`keys/sign` 也会通过——签名权限已被每次 NuGet
   发布所证明。
5. 启用签名的完整 `tauri build`。

## 故障模式与诊断

- **azure/login "federated credential not found"**：主体不匹配。打印
  任务的 `sub` 声明，与凭据主体逐字符比较——注意 2026-07-15 之后
  创建的仓库上的不可变格式（`owner@owner-id/repo@repo-id`），以及
  环境 vs ref 作用域。
- **未颁发 OIDC 令牌 / 登录未获得令牌**：任务缺少
  `permissions: id-token: write`。
- **"signCommand" 未被调用 / 二进制文件未签名**：检查配置是否落在
  `bundle.windows` 下（不是 v1 风格的顶层 `windows` 键），以及命令
  是否以 0 退出——非零退出会导致打包步骤失败。
- **找不到 AzureSignTool**：必须在运行构建的同一任务中将其作为
  `dotnet tool` 安装到运行器上。
- **来自密钥保管库的 401 / 无效令牌**：访问令牌必须作用域为
  `https://vault.azure.net`（Key Vault 资源），而不是 Azure Graph 或
  管理端点。重新检查令牌命令中的 `--resource`。
- **在新仓库中签名出现 403，但在旧仓库中正常**：意外——同一 SPN，
  同一操作。检查是否有人为新仓库创建了单独的应用注册而不是复用现有
  的；如果是，该 SPN 需要密钥保管库上的 Key Vault Crypto User。
- **本地签名有效但 SmartScreen 仍警告**：对于新应用上的 OV 证书是
  预期行为。这是信誉状态，不是缺陷——不要通过更换证书或反复重新签名
  来"修复"它。
- **更新器验证失败**：更新器签名来自 Tauri minisign 密钥对
  （`tauri signer generate`），永远不来自密钥保管库证书。如果需要
  更新器支持，请单独生成并配置它们。