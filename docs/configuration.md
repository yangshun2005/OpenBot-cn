# 配置

OpenBot 通过环境变量和租户包进行配置。API 服务器在启动时会同时校验两者。

## 环境设置

```sh
cp .env.example .env
```

填写必需的值，然后运行：

```sh
bash scripts/start.sh
```

## 必需的 API 服务器变量

| 变量                          | 含义                                                                                                  |
| ----------------------------- | ----------------------------------------------------------------------------------------------------- |
| `DATABASE_URL`                | PostgreSQL 连接字符串。                                                                               |
| `KEY_ENCRYPTION_KEY`          | 用于加密存储凭据的 Base64 编码的 32 字节密钥。使用 `openssl rand -base64 32` 生成。                   |
| `INTELLIGENCE_API_URL`        | CopilotKit Intelligence API URL。                                                                     |
| `INTELLIGENCE_GATEWAY_WS_URL` | CopilotKit Intelligence 实时网关 URL。                                                                |
| `INTELLIGENCE_API_KEY`        | Intelligence 项目的运行时密钥。                                                                       |

以上五个变量缺失时都会阻止服务器启动。三个 `INTELLIGENCE_` 值还会作为一组进行额外检查，因此部分设置会被视为配置错误而拒绝，而不是当作未配置处理。

`COPILOTKIT_LICENSE_TOKEN` 是可选的：托管的 Intelligence 不签发许可证令牌，而拥有许可证令牌的自托管 Intelligence 会设置此变量并将其转发给运行时。该 Intelligence 在其自身部署上也需要相同的许可证，否则它会拒绝记忆和其他许可功能，而对话仍可继续。OpenBot 在启动时向 Intelligence 询问一次，当答案未就绪时记录 `intelligence-entitlement-not-ready`，并指明修复方法。

`MANAGED_AGENT_AG_UI_URL` 指定了开箱即用的 Bot：在产品中创建的协作者的默认端点。它需要旁边的 `MANAGED_AGENT_TOKEN`，否则服务器拒绝启动。未设置时，服务器在没有托管 Bot 的情况下启动，随附的 Risk Analyst 协作者被省略，并且创建没有自己端点的协作者会被拒绝。有令牌但没有 URL 会被忽略。单容器镜像没有 Bot 进程，因此在那里保持 URL 未设置。`scripts/start.sh` 在笔记本电脑上将其指向 `agent-langgraph`。

## 通用变量

| 变量                 | 默认值                             | 含义                                                                |
| -------------------- | ---------------------------------- | ------------------------------------------------------------------- |
| `PORT`               | `3001`                             | API 服务器端口。`SERVER_PORT` 指定同一个端口；设置其中一个，或将两者设为相同的值，否则服务器拒绝启动。 |
| `NODE_ENV`           | 未设置                             | `production` 会拒绝示例 `KEY_ENCRYPTION_KEY`。它不决定是否需要登录；参见 `OPENBOT_SINGLE_USER`。 |
| `TENANT_PACKAGE_DIR` | `../examples/fintech`              | 租户包目录，从 `server/` 解析。                                     |
| `DEPLOYMENT_ID`      | 租户包的 id                        | 在共享的 Intelligence 项目中命名此部署，并为 Shared 应用的账户在供应商处持有的身份添加前缀。该身份在应用被设为 Shared 时固定，因此之后更改此值不会移动它。 |
| `OPENAI_API_KEY`     | 未设置                             | 内置 agent 和两个随附 Bot 的默认模型密钥。                          |
| `OPENAI_BASE_URL`    | 未设置                             | 该密钥所用于的 OpenAI 兼容端点。参见下文。                          |
| `BOT_PROVIDER`       | `openai`                           | 框架 Bot（`agent-langgraph`）和所选 harness 运行的提供商：`openai`、`anthropic` 或 `google`。Python Bot 也会读取它；`agent-bot` 不会，它仅支持 OpenAI。 |
| `ANTHROPIC_API_KEY`  | 未设置                             | 当 `BOT_PROVIDER=anthropic` 时的 Anthropic 密钥。                   |
| `ANTHROPIC_BASE_URL` | 未设置                             | 该密钥所用于的 Anthropic 兼容端点。                                 |
| `GOOGLE_API_KEY`     | 未设置                             | 当 `BOT_PROVIDER=google` 时的 Google 密钥。                         |
| `GOOGLE_GENERATIVE_AI_BASE_URL` | 未设置                   | 该密钥所用于的 Google 兼容端点。                                    |
| `BOT_MODEL`          | Bot 在规范文件中的行               | 正在启动的 Bot 的模型。未设置时，它来自该 Bot 在[提供商规范文件](#提供商规范文件)中的行；提供商回退值为 `gpt-5.5`、`claude-sonnet-4-5` 和 `gemini-2.5-flash`。 |
| `AGENT_BOT_MODEL`    | `gpt-5.5`                          | 概念验证 Bot（`agent-bot`）的模型，单独保留是因为它直接使用 `/v1/chat/completions` 并拒绝它无法使用的模型。 |
| `BOT_RESPONSES_API`  | `false`                            | 使 `agent-langgraph` 使用 OpenAI Responses API。                    |
| `BOT_REASONING_EFFORT` | 未设置（提供商默认值）           | 仅限 OpenAI 和 Responses API：`none`、`minimal`、`low`、`medium`、`high`、`xhigh`、`max` 之一。`agent-langgraph` 在任何其他值、非 `openai` 提供商或没有 Responses API 时拒绝启动。 |
| `AGENT_STALL_TIMEOUT_MS` | 未设置（关闭）                 | Bot 的流在回合被结束前可以多长时间不产生任何内容。                  |
| `AGENT_TOOL_TOKEN`   | 未设置；`start.sh` 生成一个        | 框架 Bot 通过此服务器回调已授予工具时出示的密钥。                   |
| `APP_DIST_DIR`       | 未设置                             | 当此进程提供应用时，构建后的应用所在位置。在容器镜像内设置；在开发环境中未设置，由 Vite 提供应用。 |
| `AUDIT_RETENTION_DAYS` | 未设置                           | 保留审计行的整数天数；更早的会被移除。未设置则永久保留审计追踪。    |
| `WORKER_SHARED_SECRET` | 未设置；`start.sh` 使用固定的本地默认值 | routines worker 出示以触发到期 routine 的密钥。没有它，服务器拒绝所有交接，无论是否存在发送交接的 worker。 |
| `OPENBOT_GENERATIVE_UI` | 未设置（能力开启）               | 设置为 `false` 或 `0` 以阻止 Bot 使用生成的界面进行回答。           |
| `OPENBOT_SELF_HOST_BANNER` | 未设置（横幅开启）             | 设置为 `false` 或 `0` 以为所有人隐藏提供自托管 OpenBot 帮助的栏。它在付费 Intelligence 计划上从不显示，每个人也可以为自己关闭它。 |
| `OPENBOT_ACCESSIBILITY_DISABLED` | 未设置 | `true` 或 `1` 停止在运行时已发送的分析中命名 OpenBot。              |
| `COMPOSIO_API_KEY`   | 未设置                             | 整个部署的一个密钥，用于为数百个应用持有人们账户的代理。未设置时，没有可连接的内容、没有可授予的内容，也没有 Bot 可调用的 Composio 工具；剩下的是一行无处可去的记录，位于管理 Plugins 页面上的 **More apps** 下，命名此变量。参见 [Composio](plugins/composio.md)。 |

**`OPENBOT_GENERATIVE_UI`** 默认启用生成的界面：在沙盒 iframe 中流式传输的 HTML/CSS/JavaScript，以及由 SDK 的声明式组件构建的 A2UI 界面。A2UI 按钮将其命名操作和选定值发送回当前对话的 Bot。设置 `OPENBOT_GENERATIVE_UI=false` 或 `0` 以禁用两者。`true`、`1`、空值或未设置值保持该能力开启。服务器配置两个运行时渲染器，并通过 `/api/capabilities` 向浏览器报告相同的设置。

**`OPENBOT_SELF_HOST_BANNER`** 在已登录应用顶部显示一个细栏，提供 CopilotKit 的自托管 OpenBot 帮助，链接到 `https://copilotkit.ai/talk-to-an-engineer` 并带有 `ref=openbot_app`。关闭它会保存到该人的偏好设置中，因此在他们登录的每台设备上都会保持关闭。设置 `false` 或 `0` 以为所有人隐藏它，这适合为自己的组织运行 OpenBot 的 fork。任何其他值或没有值则保持开启。

保持开启时，它在为 Intelligence 付费的部署上仍从不显示。服务器读取部署的 Intelligence 权益，并在付费计划（`pro`、`team`、`team_self_hosted` 或 `enterprise`）上激活或来自 AWS Marketplace 许可证时隐藏该栏。免费或开发者计划、未激活的权益以及无法读取的权益都会显示它。答案会保留十分钟，因此今天购买的计划会在十分钟内隐藏该栏，页面在服务器启动后最多等待一秒以获取第一个答案。在 Helm 中，通过 `config.extraEnv` 设置该变量。

组件目录有单独的按 Bot 授予。其可排序数据表（`showTable`）、交互式表单（`askForm`）以及其他编译或 playground 编写的组件仍受这些授予的约束。在 Admin → Playground 中，编辑草稿及其示例参数，预览它，然后发布以供 Bot 使用。只有已发布的代码会在对话和管理员图库中渲染；无效的 JSON 会阻止保存和发布。

生成的 HTML 在没有应用的会话或同源数据访问的情况下运行。它可以从 CDN 加载库；禁止该浏览器流量的部署可以禁用生成的 UI。

**`AGENT_STALL_TIMEOUT_MS`** 监视 Bot 具有的、追踪中其他任何内容都无法显示的故障：停止产生任何内容的流。其他每个审计行都是已发生的事情，而这一个是什么都没发生的缺失，它本身不留下任何痕迹。结束回合会写入 `agent.stream_stalled`。未设置或 `0` 会关闭它，不监视任何内容。`.env.example` 附带 `60000`，因此新克隆会开启它，而升级的部署不会未经请求地获得它。

**`AGENT_TOOL_TOKEN`** 存在是因为框架 Bot 在自己的进程中运行自己的循环，并且仍然可能无法直接访问供应商。它调用授予工具的部署，那里是授予、策略和审计行所在之处。缺失时，任何 Bot 都不得回调工具，并且会被告知这一点，而不是被悄悄允许。

该默认值对部署是正确的，对笔记本电脑是错误的，在那里它意味着每个已授予的 MCP 工具在到达授予、边界或追踪之前就被拒绝——而在那一点的拒绝在记录中不可见，因此 Bot 报告没有结果而不是错误。因此 `scripts/start.sh` 生成一个并写入 `.env`，就像它已经为 `MANAGED_AGENT_TOKEN` 所做的那样。已设置的值会保留。

它是一对中的一个，它们不可互换：`MANAGED_AGENT_TOKEN` 是服务器向 Bot 证明自己，这是 Bot 向服务器证明自己。轮换任何一个都意味着持有旧值的进程拒绝每次调用，这就是为什么 `start.sh` 在生成一个的运行中重启服务器并重新创建 Bot 容器。

**`WORKER_SHARED_SECRET`** 是另一对的相同形式的密钥：它是 routines worker 向 `/internal/routines/run` 出示以证明 routine 的调度确实来自它的内容。API 服务器在没有配置的情况下拒绝交接，而 worker 完全没有它时拒绝启动。参见 [routines.md](routines.md) 了解完全没有 worker 的部署是什么样子——当没有任何东西清扫时，Routines 页面会说明。

与 `AGENT_TOOL_TOKEN` 不同，`start.sh` 不会生成并持久化这个。它提供一个固定的本地默认值 `openbot-dev-worker-secret`，此仓库的每个克隆都得到相同的值。这在这里没问题，不是因为服务器监听的位置——它不绑定主机名，因此端口本身像任何其他端口一样可达——而是因为这是机器自己开发栈上的仅开发默认值，并且它保护的端点只接受不可猜测的 `routine_run_<uuid>` id：服务器从自己的表中重新读取 routine、所有者和频道，而不是信任调用者所说的其他任何内容，因此来自公共仓库的众所周知的值在这里不保护任何敏感内容。`AGENT_TOOL_TOKEN` 被新鲜生成并写入 `.env` 正是因为它不是那样：它被复制到每个 Bot 容器中，而持有它的框架 Bot 可能在自己的机器上运行，因此那里的固定默认值根本不会是边界。生产部署必须设置真实的 `WORKER_SHARED_SECRET`。

**`SERVER_INTERNAL_URL`** 由 worker 读取，而不是由 API 服务器读取，因此它不在上表中：它说明 worker 自己的进程可以在哪里到达此部署的 API，这是关于 worker 运行位置的事实，而不是关于 `loadConfig` 描述的部署的事实。`start.sh` 在笔记本电脑上将其指向服务器自己的端口；Helm chart 的 routines CronJob 将其指向服务器的集群内 Service 地址。

## 提供商规范文件

`shared/model-providers.json` 是一个文件，盒子中的每种语言都读取它：TypeScript Bot 通过 `shared/model-providers.ts`，Python Bot 通过 `shared/model_providers.py`，任何其他实现直接作为 JSON。唯一不这样做的是 `agent-claude-sdk`，它没有 `bots` 行。它有两个部分：每个提供商的事实，以及每个 Bot 运行的提供商和模型：

```json
{
  "providers": {
    "openai": {
      "label": "OpenAI",
      "key_variable": "OPENAI_API_KEY",
      "base_url_variable": "OPENAI_BASE_URL",
      "default_model": "gpt-5.5"
    }
  },
  "bots": {
    "agent-langgraph": { "provider": "openai", "model": "gpt-5.5" },
    "agent-adk": { "provider": "openai", "model": "gpt-4o-mini" }
  }
}
```

`BOT_PROVIDER` 和 `BOT_MODEL` 的查找顺序不变，该文件位于其中间：

1. 环境——部署如何覆盖仓库决定的内容；
2. Bot 在此文件中的行——仓库决定的内容；
3. 提供商的 `default_model`——两者都未说明时剩下的内容。

每个 Bot 保留其现有默认值：例如，`agent-mastra` 使用 `gpt-4o-mini`，而 `agent-langgraph` 使用 `gpt-5.5`。编辑 Bot 的行会更改该 Bot 的默认值，而不更改另一个 Bot 的选择。Python LangGraph harness 还接受 `BOT_PROVIDER=google_genai` 作为规范中 `google` 提供商的别名。

空值被读取为未设置，这就是 compose 文件传递 `${BOT_MODEL:-}` 给 Bot 时在没有人选择模型的情况下所给的内容。API 密钥从不出现在文件中：它们以提供商行命名的 `key_variable` 在环境中到达。

以任何语言添加 Bot 就是添加一个 `bots` 条目——`"agent-java": { "provider": "anthropic", "model": "claude-sonnet-4-5" }`——之后该 Bot 像其他所有内容一样解析其模型。命名了不存在 `providers` 条目的提供商的行，或留空字段的行，会在启动时停止每个 Bot 并给出错误键的路径，而不是在第一次模型调用时。

添加**提供商**是三个地方而不是一个：此处的 `providers` 下的行，`shared/model-providers.ts` 中 `PROVIDER_IDS` 的一个条目，以及 `shared/model_providers.py` 中 `PROVIDER_IDS` 的一个条目。每个加载器在两个方向上对照自己的列表检查此文件，因此 Python Bot 和 TypeScript Bot 都不会针对自己的加载器从未听说过的提供商行启动，也不会针对自己的加载器命名但文件没有行的提供商启动。任一拒绝都会在启动时命名错误的键，而不是在第一次模型调用时，无论哪种语言。

`agent-bot` 是唯一固定其提供商的 Bot：它直接使用 `/v1/chat/completions`，从未读取 `BOT_PROVIDER`，而 `bots.agent-bot` 仅提供其模型。

## OpenAI 兼容端点

`OPENAI_BASE_URL` 决定 OpenAI 形式的请求在哪里被应答。未设置时，就是 OpenAI。设置时，它是任何使用相同 API 的端点：位于多个提供商前面的网关、代理，或您控制的硬件上的模型。

它移动整个部署而不是一个 Bot。API 服务器为包内置 agent 读取它，`agent-bot` 为它构造的客户端读取它，而 `agent-langgraph` 为 `BOT_PROVIDER=openai` 读取它。

`OPENAI_CONTAINER_BASE_URL` 仅在 Bot 容器内覆盖该值，用于容器通过不同于主机的路由到达的端点。未设置时，容器像其他所有内容一样使用 `OPENAI_BASE_URL`。

其他两个提供商以自己的名称以相同方式工作，因为它们是不同的 API，而不是此 API 的不同 URL：`ANTHROPIC_BASE_URL` 和 `GOOGLE_GENERATIVE_AI_BASE_URL`。三者都是 API 服务器已经读取的名称，因此一行会同时移动内置 agent 和 Bot，部署不会最终有一半指向别处。

模型名称逐字传递，因此使用端点发布的任何名称。为其目录命名空间的端点需要名称的两部分，在 `BOT_MODEL` 和租户包的 `default_model` 中都是如此。

在一个密钥后面提供多个提供商的网关按通常方式寻址：

```sh
OPENAI_BASE_URL=https://gateway.internal/v1
OPENAI_API_KEY=...
BOT_MODEL=openai/gpt-5.6-terra
```

在租户包中，名称以相同方式命名空间：

```yaml
model:
  provider: openai
  credential_secret_ref: openai-api-key
  default_model: openai/gpt-5.6-terra
```

大多数网关发布模型列表，这是在配置名称之前检查名称的方式。

在将部署指向任何网关之前，有两件事值得了解。并非每个目录条目都接受工具，而没有工具调用的 Bot