# 更新日志

改了什么，写给正在决定是否升级的人。为运行 OpenBot 的人而写，而不是为写提交的人而写：当一次部署在升级后行为有所不同时，这一行就该出现在这里；如果只是代码挪了位置，就不该出现。

最新的在最前面。`Unreleased` 是 `main` 上尚未打标签的内容。

## Unreleased

**升级之前。** 对于已有的部署，有五件事会发生变化：
- 托管计算机的浏览器现在默认以有头模式运行。它比无头模式多用一点内存；设置 `COMPUTER_BROWSER_MODE=headless`（Helm：`computers.browserMode: headless`）可保留旧行为。
- 迁移 `0054_memory_on_intelligence` 为 `personal_memories` 添加两列。在拥有 Intelligence memory 权限的部署上，记忆清扫随后会把现有的、已审核且已开启的记忆复制到 Intelligence，每分钟处理几个人；该操作是幂等的，可以安全中断。等待审核、已暂停或已遗忘的记忆仍保留在本地，直到状态改变。
- 在 Admin → Automatic Learning 下未保存任何内容且未设置 `CPK_INTELLIGENCE_LEARNING_CONTAINER_ID` 时，服务器会在其 Intelligence 项目的 `openbot` 容器存在后立即分配该容器，并且 `scripts/start.sh` 会创建该容器。已保存的设置（包括关闭）永远不会被更改；`OPENBOT_LEARNING_AUTO_ASSIGN=false`（Helm：`config.learning.autoAssign: false`）可阻止此行为。
- 迁移 `0055_default_opt_outs` 添加一张表 `default_opt_outs`，用于记录某人移除默认记忆来源或后台研究的情况。随后，记忆来源和后台研究会为每个对话的人和 Bot 开启（见下文），因此预计在一个人首次与某个 Bot 对话四小时后会出现后台研究轮次及其模型 token。只有从现在起的移除才会被记录：某人在升级前删除的来源或研究设置不会留下痕迹，因此会在他们与该 Bot 的下一次对话中恢复（他们关闭并保持关闭的除外）。`OPENBOT_MEMORY_SOURCE_DEFAULTS=false` 和 `OPENBOT_PROACTIVE_DEFAULTS=false`（Helm：`config.memory.*`）可阻止此行为。
- 迁移 `0056_sign_in_code_step` 放宽了对 `sign_in_requests.status` 的检查，以允许 `code_required`（见下文）。服务器和计算机必须一起升级：旧版计算机没有 `/sign-in/code`。

### 登录的验证码在密码之后才被要求

私有登录表单过去会一次性要求用户名、密码和"验证码（如果网站要求的话）"，因此用户不得不猜测是否会有验证码，并在网站发送验证码之前就输入一个。现在表单只要求用户名和密码。如果网站在拿到密码后显示一次性验证码字段，请求会停留在新的 `code_required` 状态，表单会单独要求网站刚刚发送的验证码。验证码错误会继续要求；无人输入的验证码会随请求一起过期，用户仍可接管浏览器或取消。Bot 只会被告知结果，永远不会被告知验证码，已保存的登录也仍然不会存储验证码。该步骤会以 `computer.sign_in_code_requested` 记入审计，验证码失败则以 `computer.sign_in_failed` 并带 `step: "code"` 记录。

### 来自存储的许可证令牌现在能到达服务器

Helm chart 仅在 `secrets.licenseToken` 被写成字面量时才把 `COPILOTKIT_LICENSE_TOKEN` 传给服务器。通过 `externalSecrets` 同步的令牌（chart 自带的 EKS 示例就是这样做的）到达了 Secret，却从未到达 pod，因此自托管部署在无许可证的情况下运行，且没有任何信息说明原因。现在只要 `externalSecrets.data` 中命名了 `license-token`，它就会被传递；在 `secrets.existingSecret` 的情况下则作为可选引用传递，因为 chart 无法看到该 Secret 的键。chart 的 CI 检查现在还会在渲染结果中出现"某个存储同步了某个键但无人读取"时失败。

### OpenBot 在启动时说明 Intelligence 何时会拒绝记忆

自托管的 Intelligence 需要自己的许可证。没有许可证时，它对每次运行时权限读取都回答 `RUNTIME_ENTITLEMENTS_SELF_HOSTED_MISCONFIGURED` 并拒绝记忆，而对话继续进行，OpenBot 的日志中没有任何信息说明原因。服务器现在会在启动时询问一次，并记录 `intelligence-entitlement-not-ready`，附带代码和许可证应放置的位置；当无法询问 Intelligence 时记录 `intelligence-entitlement-unreadable`。启动不会等待该回答。

### 服务器在启动时说明其密钥何时无法调用所配置的模型

一个无权访问租户包 `default_model` 的密钥会通过所有启动检查，却在第一条消息时失败，以"The model … does not exist or you do not have access to it"的形式出现在发送者面前。服务器现在会在启动时向提供商询问一次，并记录 `model-unavailable`，附带模型和修复方法；对于被拒绝的密钥则记录 `model-credential-rejected`。它只询问 OpenAI 和 Anthropic 本身，不询问由 `OPENAI_BASE_URL` 或 `ANTHROPIC_BASE_URL` 设置的端点；未配置密钥时不作任何说明，也从不延迟启动。

### Helm chart 按摘要固定、挂载租户包，并接受 Composio 密钥

Kubernetes 上的部署过去需要手动完成的三件事现在有了各自的配置值：

- `image.digest` 精确部署某个发布版本的 `container-images.json` 所指定的镜像，格式为 `repository:tag@digest`。它会拒绝任何不是 `sha256:` 加 64 个十六进制字符的内容。
- `config.extraVolumes` 和 `config.extraVolumeMounts` 在服务器以及 routines 和 computer-culler CronJob（加载配置的 pod）上挂载卷。来自 ConfigMap 的租户包不再需要自定义镜像。
- `secrets.composioApiKey`，或在 `externalSecrets.data` 中命名的 `composio-api-key`，会作为 `COMPOSIO_API_KEY` 到达服务器。只有服务器会收到它，CronJob 不会。

在都未设置的情况下，每个目标的渲染结果与之前完全相同。

### 更少的"Verify you are human"页面

Cloudflare 背后的网站一见到 Bot 的浏览器就发起质询，打开一个页面可能会重启计算机，并给用户留下一个空白页面去解决。现在：
- 计算机不再因这类页面而重启，帮助请求会重新打开 Bot 所在的页面，因此接管的人总能拿到要解决的检查。这种交接是 Bot 无法通过的检查的设计好的回退方案。
- 托管计算机在其自己的虚拟显示上运行有头 Chromium，位于同一个实时屏幕之后。无头 Chromium 正是机器人检查最先捕获的。仍以无头模式运行的计算机不再自称 `HeadlessChrome`。
- Web Bot Auth，除非开启否则关闭：设置 `WEB_BOT_AUTH=on`（Helm：`config.webBotAuth: "on"`）并有一个 https 公共 URL 时，Bot 浏览器发送的每个请求都会被签名，密钥发布在 `/.well-known/http-message-signatures-directory`。对于允许已签名代理的网站，只需向 Cloudflare 注册一次该部署（Bot Submission Form、Request Signature）即可让 Bot 通过。参见 docs/configuration.md。Bot 访问的每个网站都会看到该部署的 URL。
- 住宅或 ISP 代理是一等设置：`EGRESS_PROXY_DEFAULT`，或在 Helm 上使用 `computers.egressProxy`，URL 放在 Secret 中。默认关闭；开启后会使 Bot 的流量看起来像个人的家庭连接，这是由运营者做出的选择。
- 全程使用 Bun 1.4.3，它修复了导致重启的连接崩溃。

### 记忆来源和后台研究默认开启

新部署过去只记住用户告诉 Bot 的内容：连接应用的记忆来源和后台研究各自等待有人逐个 Bot 设置。现在，当一个人在与某个 Bot 单独的对话中与该 Bot 交谈时，OpenBot 会为该 Bot 开启后台研究（与之前一样只读，每四小时一次，首次运行在一个间隔之后）。并且从 Bot 为某人执行的任何轮次起，该人在自己账户中连接的、Bot 可以读取的每个应用都会成为该 Bot 的记忆来源，除非是在别人发布的 Team Bot 上，此时该人自己的账户需要其同意。来源以该人的身份读取，其事实只到达该 Bot，导入的事实会等待该人审核后才进入 Intelligence，与手动创建的来源完全相同。共享应用和无需账户的应用不会被添加；用户仍需手动添加这些。

关闭其中任何一项都会保持关闭，移除也会被记住：OpenBot 永远不会重新开启它，重置 Bot 也会保留两者。已保存的任何内容都不会改变。设置 `OPENBOT_MEMORY_SOURCE_DEFAULTS=false` 或 `OPENBOT_PROACTIVE_DEFAULTS=false`（Helm：`config.memory.sourceDefaults` / `researchDefaults`）可为整个部署停止新的默认值。

### Automatic Learning 在 OpenBot 持有可用凭据的地方创建其容器

运行时密钥无法创建 Learning 容器，因此没有 `openbot` 容器的部署在有人运行辅助脚本之前什么也收集不到。现在：
- `bun scripts/setup-learning.ts`（以及每次启动时的 `scripts/start.sh`）会使用 CopilotKit CLI 登录来创建它，即使检出中未选择任何项目，也会选择运行时密钥的项目。
- `INTELLIGENCE_SERVICE_TOKEN` 配合 `INTELLIGENCE_PROJECT_ID` 让服务器自行创建它，适用于 CopilotKit 本地评估安装（这是 Intelligence 接受服务令牌用于 Learning 的唯一类型）。
- 当 `config.learning.setup` 命名了 CLI 登录 Secret 或服务令牌 Secret 时，Helm chart 会运行一个设置 Job。
- Admin → Automatic Learning 在部署持有其中之一时有一个"Create the openbot container"按钮，否则会像服务器日志一样准确说明要运行什么。

### Automatic Learning 无需访问 Admin 即可分配其容器

Learning 默认开启，但在有人分配容器之前什么也收集不到。现在，当 Admin → Automatic Learning 下未保存任何内容且未设置 `CPK_INTELLIGENCE_LEARNING_CONTAINER_ID` 时，服务器会在其 Intelligence 项目的 `openbot` 容器存在后立即将其分配为默认值，记录到审计日志中，Admin 页面也会说明。已保存的设置（包括关闭）永远不会被更改。运行时密钥无法创建容器，因此 `bun scripts/setup-learning.ts` 现在会在现有密钥的项目中创建 `openbot` 而不是拒绝，并且 `scripts/start.sh` 会在每次启动时运行该检查，无需浏览器或提示。设置 `OPENBOT_LEARNING_AUTO_ASSIGN=false`（Helm：`config.learning.autoAssign: false`）可保留旧行为。

### 记忆存放在 CopilotKit Intelligence 中

拥有 Intelligence memory 权限的部署现在将其人员的记忆保存在那里：Intelligence 保存内容并对召回进行排序，同一项目上另一个应用为某人形成的记忆会出现在其 Memory 页面并到达其 Bot。OpenBot 仍会记录每条记忆的来源、审核状态以及它来自哪个 Bot 或来源，并对 Intelligence 返回的所有内容应用这些规则，因此由 Bot 形成的记忆仍只到达形成它的那个 Bot。没有 Intelligence memory 时什么都不会改变：OpenBot 自己的表仍是存储。

## 0.2.0

**升级之前。** 对于已有的部署，有两件事会发生变化：
- 迁移 `0052_shared_brokered_accounts` 将每个 Composio 连接复制到新的 `brokered_connections` 表中，并保留 `composio_connections` 不动、不再写入。回滚到 0.1.2 可以工作，但升级后连接的账户对它不可见。
- 交接现在会记录是什么启动了它所来自的运行。较旧的远程 Bot 的已签名运行在升级后最多十分钟内不携带此类记录，在此期间它们对 Shared 应用的调用会被拒绝。

### 桌面端在每个可能编辑文件夹的命令之前都会询问

当 Bot 在从桌面应用共享的文件夹中运行命令时，用户会看到一个提示，其中可能提供"Always allow in this folder"。该选择现在也涵盖只读取该文件夹的命令。可能更改或删除其中文件的命令每次都会显示提示，且没有"Always allow"选项。已设置为"Always allow"的文件夹对只读命令的行为与之前相同。

### 一个 Bots 列表，每个 Bot 一个页面

Agents、Team Bots 和 Bots 曾是同一个 Bot 的三个列表。`/bots` 现在是唯一的列表——已固定、属于你的、与你共享的以及你隐藏的——顶部有 **New Bot**。Bot 的页面包含关于它的一切：它的 routines、它的 skills 及其可触及范围、它与谁共享，以及它的名称、角色和连接。更新到达你的位置以及你对共享 Bot 的回答已移至 Settings。`/agents` 和 `/team-bots` 仍然可用，并打开新的位置。管理员分配给某个组的 Team Bot 会固定在其成员的 Bots 列表顶部，而不是留在其侧边栏中。

侧边栏不再列出每个需要你的 Bot。Bots 上的徽章统计所有 Bot 中等待的问题、审批和停滞的交接，Bots 会将这些 Bot 列在最前面，位于 Needs you 下。当 Bot 开始需要你时的浏览器通知保持不变。

侧边栏现在是 Bots 和 Skills。每个 Bot 的页面包含该 Bot 等待你处理的事项、其建议的后续步骤、其职责、它到达你的位置、其记忆来源和后台研究，以及其审批规则。`/approvals`、`/memory`、`/reachability` 和 `/responsibilities` 仍然可用，并打开新的位置。来自 Bot、Slack 和 Teams 的、命名了这些页面的消息现在指向该 Bot 的页面。

群组对话像任何其他对话一样启动：在新对话的 To: 字段中放入两个或更多 Bot，按它们应回答的顺序排列。单独的群组按钮和 `/group/new` 屏幕已移除；`/group/new` 会打开新对话屏幕。

Settings 现在还包含你的 Bot 是否在行动前询问、每个 Bot 的规则、你的记忆、你的设备和最近的投递，以及 GitHub 事件来源。团队审批设置和共享账户请求已移至 Admin → Approvals。

### 人们可以关联他们的 Slack 和 GitHub 账户

Settings → Connected accounts 新增了一个 Linked accounts 部分，每个人可关联的账户各有一张卡片。Slack 通过发送到 OpenTag 应用的一次性代码在私信中关联；发布在频道中的代码会被取消。GitHub 通过 GitHub App 的用户登录关联，使用 `GITHUB_APP_CLIENT_ID` 和 `GITHUB_APP_CLIENT_SECRET` 设置（两者都设或都不设）；服务器在设置了它们但没有公共 URL 时拒绝启动。目前还没有任何东西读取这些关联：它们是 GitHub 警报和审查将要构建的基础。迁移 `0053_identity_links` 只添加表。

### 应用的账户可以属于团队

管理员可以将某个 Composio 应用设为 Shared：一个账户，连接一次，每个被授予该应用的 Bot 都以它行事。谁可以通过每个 Bot 使用它按 Bot 审批——仅所有者、指定的人员和组，或所有人，并单独决定是否允许电子邮件、Slack 或 webhook 输入——并在每次调用时检查。扩大它的请求会在 Approvals 收件箱中等待。通过 Shared 应用的写入会先询问用户。

### 新部署以两个同事开始，而不是十二个

示例包加载了 `examples/fintech/agents/` 中的每个同事，因此新部署一打开就有 General Assistant、Knowledge 和另外十个。这十个现在位于 `examples/fintech/catalog/`，不会被加载，新部署以 General Assistant 和 Knowledge 开始。Risk Analyst 和所选的 harness 在其端点配置后仍会注册。要保留这十个中的任何一个，请将其文件从 `catalog/` 复制到 `agents/`。

已经注册了它们的部署会保留它们。从包中移除的同事不会在下次启动时退役：其行仍保留，仍被列出，仍被删除操作以受保护为由拒绝，因为只有端点为空的远程同事才会自动退役。要从这样的部署中移除这十个，请在升级后在其 `agent_profiles` 行上设置 `deleted_at`（参见 [configuration.md](docs/configuration.md#agents)）。

## 0.1.2

### CopilotKit 1.77 和 AG-UI 1.0

OpenBot 现在运行在 CopilotKit 1.77.0（`@copilotkit/runtime`、`react-core` 和 `a2ui-renderer`，从 1.73.1 升级）上，它携带 Intelligence 客户端，并运行在 AG-UI 1.0（`@ag-ui/core` 和 `@ag-ui/client` 1.0.1，从 0.0.59 升级）上。托管 Intelligence 项目无需任何更改；移动的是服务器中的客户端。随之而来的包括：线程命名回退到第一条消息，实时连接在重复检查点之间保持其位置，中断恢复获得自己的运行 id，生成的 A2UI 控件不再提交页面自身的表单。

从自己的端点提供服务的 Bot 保持原样工作。AG-UI 1.0 仍接受较早的事件形状，因此 `agent-bot`、`agent-langgraph` 和 Python Bot 未被更改。

工具结果的 `error` 字段已从 `TOOL_CALL_RESULT` 事件中移除，因为 AG-UI 1.0 未定义该字段。拒绝仍保留在结果的文本和工具消息中，Bot 的下一轮会从那里读取它。

## 0.1.1

- 以 `-uuser:password`、`--user=user:password` 或 `-u "user:pass word"` 形式提供给 `curl` 的密码现在会在 Bot 记录的 shell 命令中被脱敏。之前只识别带空格且无引号的 `-u user:password`，因此其他写法会把密码保留在存储或导出的命令中。
- 作为 shell 命令中的 JSON 字段发送的密码、令牌或 API 密钥现在会在命令被保留或导出之前脱敏。`curl -d '{"password":"x"}'` 之前会以明文记录密钥，因为只识别 `NAME=value`