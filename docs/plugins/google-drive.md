# Google Drive

授予此连接器的 Bot 读取 Drive 的权限时，**以提问者本人的身份**进行。两个人提出同一个问题，各自得到自己账号可见的答案，且双方都看不到自己无权打开的内容。只读：请求的作用域是 `drive.readonly`，因此 Google 会在本次部署不得不处理之前就拒绝写入。

该连接器通过 Drive 的普通 REST API（`www.googleapis.com/drive/v3`）访问 Drive，而不是 Google 托管的 Drive MCP 服务器。MCP 服务器受限于 Google Workspace 开发者预览计划；其底层的 REST API 自 2015 年起就已正式可用，因此这是用"无需编写代码"换取"不依赖预览版"——对于人们所依赖的连接器而言，这是更划算的一侧。工具名称与 Google 的 MCP 服务器完全一致，因此日后换回它时，每一项授权都能保留。

设置它需要两个人，且任何一方都无法完成对方的另一半：

| 谁           | 做什么                                          | 在哪里                                    |
| ------------ | ----------------------------------------------- | ---------------------------------------- |
| 管理员       | 注册 OAuth 客户端并启用连接器                    | Google Cloud 控制台，然后是 `/admin/plugins/google-drive` |
| 每个人       | 用自己的 Google 账号进行授权同意                 | `/settings/connected-accounts/google-drive`              |

这里刻意没有提供让管理员代他人连接账号的端点。

## 管理员要做的事

### 1. 启用 Drive API

在某个 Google Cloud 项目中，启用 `drive.googleapis.com` —— 即 Drive API。这是此连接器唯一调用的 API；它通过 REST API 而非 Workspace MCP 服务器访问 Drive，因此没有第二个 API 需要开启，也无需加入任何预览计划。当凭据本身没问题却返回 `403` 时，最常见的原因就是该项目未启用此 API；参见[故障排查](#troubleshooting)。

### 2. 配置 OAuth 同意屏幕

- 作用域：`https://www.googleapis.com/auth/drive.readonly`。
- 当应用处于 **Testing**（测试）状态时，只有被列为测试用户的账号才能授权同意。所有将要连接的人都必须在该列表中，否则他们的授权会失败，并返回一个对测试用户只字未提的访问被拒错误。

### 3. 创建 OAuth 客户端

类型选择 **Web application**（Web 应用）。在 **Authorised redirect URIs**（已授权的重定向 URI）下，添加本次部署的回调地址：

```
<OPENBOT_PUBLIC_URL>/api/plugins/oauth/callback
```

在本地，它是 `http://localhost:3001/api/plugins/oauth/callback` —— 端口 3001，即 API，而不是 3010（应用）。回调落在 API 上，随后再重定向回应用。

它必须逐字符匹配：协议、主机、端口、路径，且不能有末尾斜杠。OpenBot 会在插件页面的 **Connection**（连接）部分显示要粘贴的确切字符串，该字符串由 `OPENBOT_PUBLIC_URL` 构建，而非来自传入的请求——由请求头拼装出的重定向 URI 是攻击者能插手的。请从那里复制，而不是手动输入。

保留客户端 ID 和客户端密钥，供下一步使用。

### 4. 在 OpenBot 中启用连接器

在 `/admin/plugins/google-drive`：

1. 打开 **Enable for this deployment**（为本次部署启用）。
2. 打开 **OAuth client**（OAuth 客户端），粘贴客户端 ID 和密钥。密钥使用 `KEY_ENCRYPTION_KEY` 加密，且绝不会回读到浏览器。
3. 点击 **Refresh tools**（刷新工具），它会记录此连接器实现的四个读取工具。

至此设置完成。走到这一步无需任何个人账号——此连接器的工具列表是 OpenBot 自身的代码，而非来自远程服务器的应答，因此读取它无需进行任何身份验证。

要检查它是否真的可用，请使用同一页面上的 **Your account**（你的账号）：它会连接*你的* Google 账号，然后把你带回此处。这与任何人的个人授权一样，仅能访问你自己的文档，且不属于配置连接器的一部分——无论管理员是否连接过，部署的配置都是正确的。

### 5. 向 Bot 授予工具

启用连接器并不会让任何 Bot 获得对它的访问权限。每个工具都按 Bot 逐一授予，与其他所有插件工具一样。此后每次调用都会检查授权、评估操作策略，并写入一行审计记录。

## 每个人要做的事

在 `/settings/connected-accounts`，一旦管理员启用了它，Google Drive 就会出现。打开它并点击 **Connect**（连接）。这会离开 OpenBot 前往 Google 自己的同意屏幕——按钮上的箭头已表明这一点——然后返回同一页面，此时页面会显示 **Connected**（已连接），并附上 Google 实际授予的作用域。

没有任何缓存。OpenBot 存储刷新令牌，并为每次调用铸造一个短期访问令牌，因此在 Google 处撤销访问会在下一次调用时生效，而不是等到某个缓存过期。

### 断开连接

**尚未构建。** 在它构建出来之前，请在 Google 自己的第三方访问设置中撤销（[myaccount.google.com/connections](https://myaccount.google.com/connections)），这会立即阻止本次部署读取任何内容。页面也是这么说的，而不是提供一个会在访问并未撤销时却报告已撤销的控件。

## 故障排查

下面的每一条消息都是 OpenBot 实际显示的。它们值得逐字阅读：连接器区分"凭据被拒绝"与"凭据被接受但请求被拒绝"，而这两者的修复方法完全不同。

**先看审计追踪。** 每次调用都会留下一行记录，在尝试之后写入，而事件类型就是"这是谁的问题"的答案：

| 记录行               | 含义                                                                  |
| -------------------- | ---------------------------------------------------------------------- |
| `mcp.call_rejected`  | 本次部署拒绝了。缺少授权，或某条策略规则——`decision.rule` 会指明是哪一条。 |
| `mcp.call_failed`    | 在此处被允许，但在供应商处失败。`failure` 携带供应商自己的原话。 |
| `mcp.call_succeeded` | 供应商已应答。                                                    |

一个看似没有访问权限、且**完全没有留下任何记录行**的 Bot，从未调用过该工具，这是授权问题而非连接问题：请检查该工具是否已授予*那个* Bot，在 `/admin/plugins/google-drive`。即便已启用连接器并连接了你的账号，每个工具仍然处于未授予状态。

### 同意屏幕上出现 `redirect_uri_mismatch`

Google 正在将 OpenBot 发送的 `redirect_uri` 与 OAuth 客户端上的列表进行精确字符串比对。请将插件页面 **Connection** 下显示的值与已注册的值逐字符比较。常见的不匹配：`127.0.0.1` 与 `localhost`、用了应用的端口而非 API 的端口、`https` 与 `http`、末尾斜杠。

错误所涉及的客户端，就是其 ID 出现在 URL 中的那个。拥有多个 Google 客户端的部署，可能把 URI 注册到了错误的那个上。

### "Google Drive refused this request (401)."

Google 完全不接受该令牌。在 `/settings/connected-accounts/google-drive` 重新连接账号是通常的修复方法。如果问题持续，则说明授予的作用域未覆盖此连接器——请将 **Access**（访问）部分报告为已授予的内容与此连接器请求的 `drive.readonly` 作用域进行比对。

### "Google Drive refused this request (403): …"

凭据没问题；Google 接受了它，但拒绝了请求。请阅读冒号后的那句话——那是 Google 自己的原话，对于最常见的原因，它会指明该 API 并附上用于启用它的控制台 URL。那个原因就是[第 1 步](#1-enable-the-drive-api)：项目未启用 `drive.googleapis.com`。

启用 API 需要几分钟才能生效。请稍等，然后重试。

如果那句话是关于对某个特定文件的访问而非某个 API，则说明该账号确实看不到所请求的内容，这正是连接器按预期工作。

### "Google Drive could not be reached: …" 或 "Google Drive did not answer in time."

不是凭据问题：请求从未从 Google 得到裁决。前者是网络或 DNS 故障，并携带其自身的原因；后者是请求超出了连接器的超时时间。两者都值得重试，而持续出现则关乎本次部署与 Google 之间的链路，而非账号。

### "You have not connected your Google Drive account."

Bot 被要求以某个未存储连接的人的身份读取。请在 `/settings/connected-accounts/google-drive` 连接。这里没有回退到部署级凭据的机制——这是刻意设计的，因为回退会用别人的访问权限来作答。

### 连接成功，约一小时后失效

那是一个背后没有刷新令牌的访问令牌，OpenBot 拒绝存储它正是为了防止这种情况发生；如果你看到了，请报告，因为这意味着有东西绕过了那道检查。当 Google 认为此人已经授权同意时，它不会返回刷新令牌，这就是为什么授权 URL 同时发送 `access_type=offline` 和 `prompt=consent`。

### "The tool returned no content. Nothing was found, so there is nothing here to answer from."

不是错误。工具运行了，但没有匹配到任何内容，这句话的存在是为了告知模型这一点，而不是递给它一个空字符串让它凭记忆去填充。

## 另见

- [架构](../architecture.md) —— 插件、授权、策略和审计所处的位置。
- [配置](../configuration.md) —— `OPENBOT_PUBLIC_URL`、`OPENBOT_APP_URL`、`KEY_ENCRYPTION_KEY`。
- [Google Drive API](https://developers.google.com/workspace/drive/api/reference/rest/v3) —— 此连接器所调用的 REST API。