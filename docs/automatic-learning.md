# 自动学习

打开 **Admin → Automatic Learning**，选择哪些 Bot 将已完成的对话贡献给 CopilotKit Intelligence Learning 容器，并接收其发布的技能。对于托管部署和自托管部署，学习功能默认开启；已保存的关闭设置会被保留。收集和交付需要存在一个已分配的容器。即使没有容器，OpenBot 仍可启动和聊天。

开箱即用，OpenBot 会自行分配该容器：当 Admin 中未保存任何内容且环境中未指定容器时，服务器会在其 Intelligence 项目的 `openbot` 容器存在后立即将其设为默认容器。参见[自动分配](#automatic-assignment)。

桌面端登录会创建或复用所选项目的 `openbot` 容器。托管设置使用 CopilotKit 登录；自托管设置使用该部署的正常浏览器登录，并要求已安装 Chrome 或 Edge。仅当预置的密钥和 API 仍然匹配时，Start 才会保存分配，并保留 `.env` 中显式的自定义容器。已保存的 Admin 设置（包括关闭或空默认值）仍然具有权威性。

对于全新的托管克隆，将 `.env.example` 复制为 `.env`，然后在 OpenBot 根目录运行：

```sh
bun install
npx --yes copilotkit@latest login
npx --yes copilotkit@latest project select
bun scripts/setup-learning.ts
```

该辅助脚本为所选项目预置密钥，创建或复用其 `openbot` 容器，并在写入 `INTELLIGENCE_API_KEY` 和 `CPK_INTELLIGENCE_LEARNING_CONTAINER_ID` 之前验证响应。它会保留自定义目标，且绝不替换现有的 OpenBot 运行时密钥：如果已设置密钥，它只会确保该密钥的项目中存在 `openbot`（见下文）。

对于全新的自托管设置，将 `INTELLIGENCE_API_URL` 设置为该部署的 HTTPS API 源，并在 `bun install` 之后运行 `bun scripts/setup-learning.ts`；跳过托管 CLI 命令。请先安装 Google Chrome 或 Microsoft Edge。该辅助脚本会为你的部署登录打开一个隔离的浏览器，列出你可访问的项目，并在为所选项目铸造密钥之前创建或复用 `openbot`。它仅在该临时浏览器中保留登录 Cookie，并在完成、取消或超时时关闭它。HTTP 仅支持回环开发。请在浏览器能够访问 Intelligence 的计算机上运行此运维设置，包括准备远程 Web 或 Helm 部署时；使用你正常的密钥管理方式传输生成的运行时密钥，并将容器 ID 设置为 `openbot`。

仅凭运行时密钥无法创建容器：Intelligence 仅向个人会话或 CopilotKit CLI 登录提供容器创建服务。对于已有密钥的部署，`bun scripts/setup-learning.ts` 会询问该密钥其项目是否有 `openbot`。如果没有，辅助脚本会使用你的凭据创建它：在托管 Intelligence 上通过 CLI 创建，但仅在创建任何内容之前确认 CLI 登录的组织和项目属于该密钥（组织按名称确认，因为 CLI 不报告 slug；项目按 slug 和名称确认，若已选择则按所选项目确认）；在自托管 Intelligence 上通过与全新设置相同的浏览器登录创建，而不铸造新密钥。无论哪种方式，密钥本身随后必须能看到该容器，否则一切都不算数。辅助脚本不会向 `.env` 写入任何内容；服务器会自行分配容器。`scripts/start.sh` 在每次启动时以 `--non-interactive` 运行相同的检查，因此带有 CLI 登录的检出无需任何人运行辅助脚本即可获得容器。该模式从不打开浏览器、提示或预置密钥，也从不阻止 OpenBot 启动。

自定义容器分配以及无法运行辅助脚本的部署，请使用经过身份验证的 Intelligence 管理功能和以下手动步骤：

1. 在 OpenBot 使用的同一 Intelligence 项目中，创建一个 Learning 容器。其稳定 ID 包含 1–64 个小写字母、数字和单个连字符（例如 `support-learning`）。
2. 在 OpenBot 中，输入默认容器 ID，并在之前暂停的情况下重新启用 Learning。将默认值留空则仅选择特定的 Bot。每个 Bot 可以继承默认值、使用另一个容器或被排除。保存设置。
3. 启动**新线程**并完成相关工作流。聊天、频道、计划例程和交接使用相同的服务器端设置。现有线程保留首次容器分配（包括未分配的首次运行）；更改 Bot 映射不会移动或回填其先前的证据。
4. 在 Learning 页面使用 **Manage Learning in Intelligence**、**View analysis runs** 和 **Review skill candidates**。Intelligence 收集证据、运行分析并提出技能。在 Intelligence 中审查源证据并发布已批准的修订。计划分析或启用 Learning 都不会自动批准技能。
5. 在 Intelligence 中为该容器启用技能交付，并启动另一次 Bot 调用。已发布的指导会加载到 Bot 的受治理运行中。精确的修订固定会保留特定的已发布修订；将其留空则在刷新窗口后跟随新发布。仅加载快照并不能证明模型使用了技能：还要检查其技能工具调用。

对于与启用了 Learning 的 Bot 进行的实时语音通话，实质性回答、解释、建议、研究和操作会通过现有的 `ask_agent` 频道运行进行委托。该所选 Bot 的 AG-UI 运行和 Rich Threads 提供 Learning 证据和已发布指导。简短的口头确认、澄清以及朗读或总结其返回的答案仍为音频，不构成证据。语音会话在启动时读取参与状态：更改 Learning 设置后请重启通话。技能快照和 Intelligence 项目密钥都不会发送给实时语音提供商。

暂停 OpenBot Learning 会保留其映射和线程分配。它会停止新的 OpenBot 分配和交付。先前绑定的线程仍保留在其 Intelligence 容器中；在 Intelligence 中暂停分析以停止从这些容器进行学习。交付在全新调用时使用当前的每 Bot 策略。Intelligence 现有的证据、计划分析、发布和单独的交付控制仍在那里管理。

## 自动分配

在未分配容器的情况下，服务器使用其运行时密钥询问 Intelligence，该项目是否有 ID 为 `openbot` 的容器。如果有，服务器会将其保存为默认值，与管理员在 Admin 中保存完全一样，并记录一行由该部署发起的 `configuration.changed` 审计记录。它在启动时、每十分钟以及当一轮对话或 Admin 页面读取 Learning 设置时询问，最多每分钟一次。稍后创建的容器无需重启即可被拾取。

- 它绝不覆盖选择。任何已保存的 Admin 设置（包括关闭或空默认值）都意味着无事可做，`CPK_INTELLIGENCE_LEARNING_CONTAINER_ID` 也是如此。
- 它仅使用能够创建的凭据来创建容器，绝不使用运行时密钥。参见[创建容器](#creating-the-container)。
- 它在多个副本上是安全的。保存仅在从未保存过任何内容的地方成功，因此一个副本胜出并写入审计记录。
- 它绝不阻止启动或一轮对话。缺少容器、没有 Learning 的计划或无法访问的 Intelligence 会作为一行 `learning-auto-assign` 记录一次，并在稍后重试。Admin 页面会说明适用哪一种。
- `OPENBOT_LEARNING_AUTO_ASSIGN=false`（Helm：`config.learning.autoAssign: false`）将其关闭。

## 创建容器

Intelligence 仅为三类调用者创建 Learning 容器（`POST /api/learning/projects/:projectId/containers`）：个人浏览器会话、CopilotKit CLI 登录和服务令牌，后者仅在 CopilotKit 本地评估安装中有效。项目的运行时密钥不属于其中任何一类。因此，OpenBot 在持有其中一种凭据的地方创建 `openbot`，用运行时密钥证明结果（密钥看不到的容器在另一个项目中，会被报告，绝不分配），否则会准确说明要运行的内容：

| 位置 | 凭据 | 发生的情况 |
| --- | --- | --- |
| `bun run start` / `scripts/start.sh` | 你的 CLI 登录（`npx --yes copilotkit@latest login`） | 每次启动运行 `scripts/setup-learning.ts --non-interactive`。如果检出中未选择项目，它会列出你的登录可看到的项目，并使用运行时密钥的项目，从临时目录运行，因此检出中没有任何内容被选择或重写。如果机器上没有登录，它不会下载 CLI。 |
| `bun scripts/setup-learning.ts` | CLI 登录（托管）或浏览器登录（自托管） | 如上所述，以交互方式进行；自托管会打开浏览器。 |
| 服务器，在启动时和从 Admin | `INTELLIGENCE_SERVICE_TOKEN` + `INTELLIGENCE_PROJECT_ID`（本地评估安装），或同一机器上的 CLI 登录 | 如果缺失，在启动时创建一次。当存在其中之一时，**Admin → Automatic Learning** 会显示 **Create the openbot container**，以便重试。 |
| Helm | `config.learning.setup.cliLoginSecret` 或 `serviceTokenSecret` + `projectId` | 每个发布修订版本运行一个 Job 执行 `server/src/learning/ensure-container.ts` 并退出；服务器在下次检查时分配容器。CLI 登录仅由该 Job 使用。 |
| 桌面应用 | CopilotKit 或自托管登录 | 连接会创建或复用所选项目的 `openbot`（见上文）。 |

如果这些都没有，`learning-auto-assign` 日志行和 Admin 页面会说明你的 Intelligence 的确切步骤：对于托管，使用 CLI 登录并运行辅助脚本（或为 chart 提供 CLI 登录）；对于自托管，运行辅助脚本并登录，或在本地评估安装上设置服务令牌。在 Intelligence 中手动创建 ID 为 `openbot` 的容器在任何地方都有效。

## 凭据和权限

OpenBot 复用其**服务器端**的 `INTELLIGENCE_API_KEY` 和 `INTELLIGENCE_API_URL`。设置使用经过身份验证的账户创建容器，而不是运行时密钥。自动分配仅使用运行时密钥读取其自身项目中是否存在 `openbot`。切勿将项目密钥放入浏览器环境变量。Learning 设置和检查端点需要 OpenBot 管理员；设置写入还需要同源 JSON 请求（或明确受信任的应用源）。

项目密钥支持运行时摄取、技能交付和只读 Inspector Learning 投影。它不是管理凭据。审查、运行、计划和发布链接会打开现有的 Intelligence UI，你在那里使用有权访问该项目的账户登录。OpenBot 不会使用其项目密钥代理管理写入。

Admin 页面报告证据和已发布技能计数、分析状态以及可用的交付诊断。不可用状态会显示为带重试操作的错误，而不是空容器。服务器错误和项目凭据不会在这些消息中暴露。

## 托管和自托管部署

对于托管 Intelligence，保留现有的托管 API 和网关配置。在其关联项目中配置容器。对于 Enterprise 自托管 Intelligence 部署，使用其 API 和网关端点以及同一项目的密钥。在 OpenBot 中分配容器之前，安装支持 Automatic Learning、Inspector Learning 端点（`GET /api/inspector/learning`）和已发布技能交付的 Intelligence 版本和迁移。链接来自该服务器配置的 Web 应用源，因此自托管管理会打开正确的 UI。

OpenBot 的数据库迁移添加了 `learning_settings` 和 `learning_thread_bindings`；运行正常的部署迁移。设置从数据库跨副本读取，线程分配是原子的。

运维人员可以在 API 服务器环境中设置初始默认值：

```dotenv
CPK_INTELLIGENCE_LEARNING_CONTAINER_ID=support-learning
# Optional opaque published revision ID; omit to follow latest.
CPK_INTELLIGENCE_SKILLS_REVISION=exact-revision-id
```

这些是可选的默认值，不是额外的启动要求。一旦管理员保存设置，这些已保存的设置优先，包括已保存的关闭状态。Helm 将默认值公开为 `config.learning.containerId` 和 `config.learning.revision`。本地开发从 `.env` 读取它们；桌面安装可以使用 Admin 页面而无需修改原生配置。修订版本是不透明的 ID，不是要递增的数字或特殊的 `auto` 值。

内置代理通过 CopilotKit 运行时接收已发布指导。OpenBot 随附的远程代理在每次运行时接收服务器选择的、经过验证的技能快照；它们不需要 Intelligence 项目密钥。自带远程代理必须消费该转发的上下文（或配置框架支持的本地适配器）才能使用交付的指导；仅收集证据不会改变第三方代理的提示。保留现有的工具授权、边界、凭据和手动编写的斜杠技能：学习到的指导不会授予工具或替换它们。

有关当前的部署要求和审查行为，请参阅权威的 [Automatic Learning 指南](https://docs.copilotkit.ai/learning)和[技能交付指南](https://docs.copilotkit.ai/intelligence/learned-skills)。