

> **[中文化] OpenBot**
>
> 此项目是 [CopilotKit/OpenBot](https://github.com/CopilotKit/OpenBot) 的中文翻译版本。
> - 原项目 Stars: 6438
> - 主语言: TypeScript
> - 许可证: MIT
> - 翻译日期: 2026-10-14
> - 原始 README: [README_en.md](README_en.md)
>
> 说明：下方 README 正文为**原项目文档的中文翻译**，力求忠于原文；本页顶部的「中文快速上手」为**面向中文用户的补充导读**（非原文内容），目的是让你不用读完整篇 README 就能先把项目跑起来。
>
> 如有翻译不准确之处，欢迎提 Issue 或 PR。

---


## 🚀 中文快速上手（补充导读，非原文）

### 这是什么 / 解决什么问题

OpenBot 是一个运行在你自己基础设施内部的 AI 助手平台，形态类似 ChatGPT / Claude / Grok，但可以完全由你掌控和修改。它通过 AG-UI 协议接入任意智能体技术栈，每个 Bot 拥有独立的浏览器、登录凭据和文件，且所有操作在执行前经过网关决策、执行后被记录。它解决的是「想用 AI 助手但数据/工具必须留在自己环境里」的问题。注意：这是一个模板而非产品，没有托管版本，需要你克隆仓库后自行改造。

### 快速上手（最短路径）

1. 创建 `.env` 并安装依赖：

   ```sh
   cp .env.example .env
   bun install
   ```

2. 连接一个托管 Intelligence 项目：

   ```sh
   npx --yes copilotkit@latest login
   npx --yes copilotkit@latest project select
   bun scripts/setup-learning.ts
   ```

3. 填写其余必需的值：`OPENAI_API_KEY`（如自建 Intelligence 则另需相关配置）。如需自定义加密密钥：

   ```sh
   openssl rand -base64 32
   ```

4. 启动：

   ```sh
   bash scripts/start.sh
   ```

5. 打开 <http://localhost:3010>。

（停止用 `scripts/stop.sh`，加 `--keep-computers` 可保留 Bot 的计算机。）

### 环境要求

- **操作系统**：README 面向笔记本电脑编写；本地运行 Intelligence 的预览版面向 macOS 配合 Docker Desktop。
- **硬件**：原文未说明具体规格（部署最低规格见 `docs/deployment.md`，材料未展开）。
- **依赖**：
  - Docker（用于 PostgreSQL 和随附的 Bot）
  - Bun 1.3+（用于应用和 API 服务器；`package.json` 中 `packageManager` 为 `bun@1.4.3`）
  - 一个 CopilotKit Intelligence 项目和许可证（有免费计划，也可自托管）
  - 一个模型密钥（概念验证 Bot 用 OpenAI；LangGraph Bot 可用 OpenAI、Anthropic 或 Google）
- **版本要求**：Bun 1.3+；Dockerfile 中 Playwright 版本为 1.62.1，需与 `agent-computer/package.json` 保持一致。
- **国内网络提示**：材料未提及镜像源或加速下载相关内容，原文未说明。

### 常见坑与注意事项

- **这是 Alpha 阶段模板，不是产品**：没有托管版本可注册，仓库内每个工作区都是私有的，不发布为可依赖的包，需要你克隆后自行改造。
- **默认单用户放行**：`.env.example` 中 `OPENBOT_SINGLE_USER=true` 会把每个请求当管理员放行，全新克隆无需 OAuth 即可访问；但**在其他人能访问部署之前必须配置登录**，否则不安全。
- **需要外部密钥与项目**：必须提供 CopilotKit Intelligence 项目/许可证和一个模型密钥（如 `OPENAI_API_KEY`）；自托管 Intelligence 时还需手动设置 `INTELLIGENCE_GATEWAY_WS_URL`，且缺少 `INTELLIGENCE_API_KEY` 会拒绝启动。
- **本地 Intelligence 是预览版**：仅面向 macOS + Docker Desktop，本地许可证有效期 30 天，不是生产安装；自托管设置需先安装 Chrome 或 Edge 浏览器。

---



---

<div align="center">

# OpenBot

**你的公司真正能够拥有的 AI 助手。** 形态与 ChatGPT、Claude 或 Grok 相同，但有一个关键区别：它运行在你自己的基础设施上，你可以随意修改它的任何部分。任何智能体技术栈，通过 AG-UI 接入。

每位同事都拥有自己的计算机：一个带有独立登录凭据、独立文件的真实浏览器，且只能使用你授予的工具。每一个操作在执行前经过决策，在执行后被记录。

[**与工程师交流**](https://copilotkit.ai/talk-to-an-engineer?ref=openbot_readme) · [**让我们与你一起构建**](https://copilotkit.ai/talk-to-an-engineer?ref=openbot_readme) · [**copilotkit.ai/openbot**](https://copilotkit.ai/openbot) · [**快速开始**](#quick-start) · [**文档**](docs/README.md)

[![CI](https://github.com/CopilotKit/openbot/actions/workflows/ci.yml/badge.svg)](https://github.com/CopilotKit/openbot/actions/workflows/ci.yml)
[![security](https://github.com/CopilotKit/openbot/actions/workflows/security_zizmor.yml/badge.svg)](https://github.com/CopilotKit/openbot/actions/workflows/security_zizmor.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
![Alpha](https://img.shields.io/badge/status-alpha-orange.svg)

[![Trendshift: #3 Repository Of The Day](https://trendshift.io/api/badge/trendshift/repositories/175080/daily)](https://trendshift.io/repositories/175080)

</div>

https://github.com/user-attachments/assets/535ef7ee-1631-4a69-b839-564c56cf90b4

<div align="center">

接入任何 AG-UI 智能体，无论是基于框架构建还是手写实现，它都会作为一个拥有独立频道的同事出现。在它自己的屏幕上观看它工作，当它遇到不应独自处理的事情时接管控制，然后再交还给它。它以组件而非仅以文字来回应，整个系统运行在你自己的机器上。

</div>

> **这是一个模板，而非产品。** OpenBot 的设计意图是被克隆并改造成你自己的东西。没有可供注册的托管版本，这里也没有任何以包的形式发布供依赖的内容：本仓库中的每个工作区都是私有的。你获取这个仓库，将 `examples/` 下的示例租户包替换为你自己的同事、频道和技能，然后运行它。以下所有内容描述的是一个起点，而非某个已完成的、由他人为你运营的东西。

> **Alpha 阶段，正在积极开发中。** OpenBot 还很早期。请预期会有粗糙的边缘和 bug，也请预期会有变化。欢迎提交 Issue 和 Pull Request。

> **运行在你的机器上。** 以下所有内容都是为笔记本电脑编写的。`.env.example` 中带有 `OPENBOT_SINGLE_USER=true`，它会将每个请求都作为一个管理员放行，因此全新克隆的仓库无需先注册 OAuth 客户端即可访问产品。[登录](#sign-in)会关闭该设置，且在其他人能够访问部署之前必须配置。

> **我们可以在你的公司为你构建，或与你一起构建。** 我们的工程师会在你的基础设施内部搭建 OpenBot，将其定制成看起来像你自己产品的样子，然后交还给你继续修改。[**开始对话**](https://copilotkit.ai/talk-to-an-engineer?ref=openbot_readme)。
>
> 与此同时，你可以在笔记本电脑上运行全部内容，包括 Intelligence：CopilotKit 的免费 Developer 计划涵盖在 Mac 上通过 Docker [在本地运行 Intelligence](#intelligence-on-your-own-machine)。

## 它是什么

一个运行在你自己的基础设施内部的智能体平台。Docker Compose 启动它的每一个部分，数据存放在你的 PostgreSQL 中，模型由你选择：开箱不附带任何模型，由管理员提供凭据，凭据在静态时加密且从不记录日志。

一次部署以两个同事开始，它们是配置而非代码：**General Assistant** 用于日常工作，**Knowledge** 用于公司问题。一旦配置好端点，**Risk Analyst** 会加入它们，另有十个在 `examples/fintech/catalog/` 中等待，每个只做一件事——按照成文政策审阅费用报销、将会议记录转化为其中实际包含的后续事项、根据已发布的内容起草发布说明、对工单进行分诊、根据手册回答新员工的问题、撰写一份简报并指出未能找到的内容、整理面试记录、交接值班班次、在续约前汇总已知信息、将客户反馈归类为可引用的主题。将其中的一个复制到 `examples/fintech/agents/` 即可添加它，在那里或 `agents.yaml` 中编写你自己的，或者从 UI 的 `/bots` 创建一个。

Bot 对计算机、文件、MCP 服务器或组件所做的任何操作都经过一个网关来决策和记录。这就是能够使用你的工具的智能体与你可以放心让它接近这些工具的智能体之间的区别。

更多内容见 [copilotkit.ai/openbot](https://copilotkit.ai/openbot)。

## 基于 AG-UI 构建

Bot 是任何使用 [AG-UI](https://github.com/ag-ui-protocol/ag-ui) 协议的端点，AG-UI 是智能体与用户交互的开放协议，因此 OpenBot 不绑定于任何框架，你也不受此限制。使用 LangGraph、Mastra、CrewAI、Pydantic AI、Google ADK 构建的智能体或手写的智能体都以相同方式接入，治理机制依托协议而非框架。

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/architecture-dark.svg">
  <img src="assets/architecture-light.svg" alt="你与服务器对话，服务器通过 AG-UI 将该轮次发送给 Bot。Bot 发起的每次工具调用都通过网关返回，网关解析目标、根据你的策略进行决策、记录审计行，然后才执行操作，或拒绝并指出所依据的规则。被允许的浏览器和文件操作到达该 Bot 自己的计算机，每个容器都有自己的 Chromium、登录凭据和工作区，由 supervisor 构建。决策落入 PostgreSQL，线程落入 CopilotKit Intelligence。">
</picture>

## 环境要求

- Docker，用于 PostgreSQL 和随附的 Bot。
- [Bun](https://bun.sh) 1.3+，用于应用和 API 服务器。
- 一个 CopilotKit Intelligence 项目和许可证。有免费计划可用，Intelligence 也可以自托管。
- 一个模型密钥。概念验证 Bot 使用 OpenAI；LangGraph Bot 可以使用 OpenAI、Anthropic 或 Google。

## 快速开始

> **使用 AI 助手进行设置？** 先将 [`prompt.txt`](prompt.txt) 粘贴给它。它包含与下方相同的步骤，以及容易出错的事项：`.env.example` 中哪些空白键实际上需要你填写（CopilotKit 密钥和一个模型密钥），哪些由启动脚本为你填写，以及每种启动拒绝的含义。其中的每一项声明都已对照本仓库进行核实。

1. 创建 `.env`：

   ```sh
   cp .env.example .env
   bun install
   ```

2. 连接一个全新的托管 Intelligence 项目：

   ```sh
   npx --yes copilotkit@latest login
   npx --yes copilotkit@latest project select
   bun scripts/setup-learning.ts
   ```

   该辅助脚本会为所选项目配置运行时密钥，创建或复用其 `openbot` Learning 容器，然后将这两项设置写入 `.env`。它会保留自定义容器分配，且从不替换已有的 OpenBot 密钥；如果已设置了一个密钥，它只会在该密钥所属项目中创建缺失的 `openbot`。
   对于全新的自托管设置，将 `INTELLIGENCE_API_URL` 设置为该部署的 HTTPS API 源（本地部署则为 `http://localhost`），并在不运行那两个 `npx` 命令的情况下运行辅助脚本。它会打开一个隔离的 Chrome 或 Edge 窗口，供你进行常规登录和项目选择；请先安装其中任一浏览器。它写入 API URL、密钥和容器，但不写入 `INTELLIGENCE_GATEWAY_WS_URL`，因此你需要自行将该项设置为同一部署。一旦 `openbot` 存在，服务器会自动将其分配为默认值，除非 [Admin → Automatic Learning](docs/automatic-learning.md) 中有已保存的设置。
   托管 Intelligence 不需要单独的许可证令牌。

3. 填写其余必需的值：

   - `OPENAI_API_KEY`

   除非你自己运行 Intelligence，否则保留 `.env.example` 中的托管 Intelligence URL。示例中的 `KEY_ENCRYPTION_KEY` 是公开的，在本地使用没问题；用以下命令生成你自己的：

   ```sh
   openssl rand -base64 32
   ```

4. 运行：

   ```sh
   bash scripts/start.sh
   ```

5. 打开 <http://localhost:3010>。

`scripts/start.sh` 会启动 Docker 服务、应用迁移、在端口 3001 上启动 API 服务器、启动例行任务 worker、在端口 3010 上启动应用，并在打印后续步骤之前检查各服务是否响应各自的健康检查路由。

`scripts/stop.sh` 会关闭相同的内容，包括每个 Bot 的计算机（compose 不管理它们）；`--keep-computers` 会让计算机保持运行。不会删除任何内容：数据库、Bot 的文件及其浏览器配置文件都是卷。

### 在你自己的机器上运行 Intelligence

要在单台 Mac 上试用 Intelligence 而非使用托管服务，请遵循[在本地评估 Intelligence](https://docs.copilotkit.ai/intelligence/self-hosting-local)。免费的 [Developer 计划](https://www.copilotkit.ai/pricing)符合条件。本地许可证有效期为 30 天，只要你的账户状态良好，`npx copilotkit@latest local renew` 可以按需续期。这是面向 macOS 配合 Docker Desktop 的预览版，不是生产安装。在 OpenBot 根目录（即 `.env` 所在位置）运行其 `npx copilotkit@latest local connect`。

该命令会将 `INTELLIGENCE_API_URL`、`INTELLIGENCE_GATEWAY_WS_URL` 和 `CPK_INTELLIGENCE_API_KEY` 写入 `.env`。OpenBot 按原样读取前两项，但只从 `INTELLIGENCE_API_KEY` 获取其项目密钥，且缺少它时会拒绝启动，因此在 `bash scripts/start.sh` 之前将 `cpk-...` 的值复制过去：

```sh
INTELLIGENCE_API_KEY=cpk-...   # CPK_INTELLIGENCE_API_KEY 的值
```

要在该技术栈上为 Bot 提供 Learning 容器，请参阅[自动学习](docs/automatic-learning.md)。

## 部署它

一个镜像包含应用、API、Bot 驱动的浏览器，以及可选的 PostgreSQL。相同的 `.env`，无需 Kubernetes。

```sh
# 已发布的镜像。无需克隆，无需构建。
docker run -p 3001:3001 --env-file .env \
  -e EMBEDDED_POSTGRES=on -v openbot-data:/var/lib/postgresql \
  ghcr.io/copilotkit/openbot:latest

# 或者你面前的这棵代码树。
docker build -t openbot .
docker run -p 3001:3001 --env-file .env \
  -e EMBEDDED_POSTGRES=on -v openbot-data:/var/lib/postgresql openbot
```

这里所有内容都在 3001 上，包括应用，而非克隆时使用的 3010。`latest` 是最新的发布版本，像 `:v0.0.9` 这样的版本标签会固定到某一个版本。

不设置 `EMBEDDED_POSTGRES`，并将 `DATABASE_URL` 设置为你已运行的数据库。[docs/deployment.md](docs/deployment.md) 中有最低规格、平台注意事项，以及它在多个副本后面的行为方式。

## 试用它

- 打开 `/bot` 并提问：`Open news.ycombinator.com and tell me the top story.`
- 让 Bot 填写 <https://httpbin.org/forms/post>，然后检查 `/admin/audit`。
- 打开 `/admin/boundaries`，添加一条拒绝规则或预设，然后重试相同的浏览器操作。
- 从 `/bots` 创建一个 Bot，给它一个常设角色，并与它开始一个频道。

## 主要界面

| 路由                        | 用途                                                               |
| --------------------------- | ------------------------------------------------------------------ |
| `/`                         | 开始和浏览频道。                                                   |
| `/channel/new`              | 开始一段对话；在收件人中放入两个或更多 Bot 即可开始群组。          |
| `/channel/:id`              | 与一位同事对话，观看它的屏幕，查看它运行了什么。                   |
| `/bot`                      | 与 Bot 直接聊天；`?agent=<id>` 选择一个。                          |
| `/bots`                     | 你可以访问的每个 Bot；创建一个，并打开其页面进行修改。             |
| `/bots/:id`                 | 单个 Bot：它等待你做什么、它正在做什么，以及它是如何设置的。       |
| `/bots/:id/responsibilities` | 给 Bot 一个长期目标，跟踪其进展，并决定它何时应该工作。            |
| `/bots/:id/reach`           | 在 Slack、Microsoft Teams 或通过短信继续与 Bot 的对话。            |
| `/bots/:id/memory`          | 哪些已连接的应用为 Bot 提供事实，以及它的后台研究。                |
| `/bots/:id/approvals`       | Bot 可以不先询问你就做什么。                                       |
| `/skills`                   | 创建并启用个人技能。                                               |
| `/routines`                 | 查看常设的例行任务，并停止其中一个。                               |
| `/settings`                 | 用户偏好设置。                                                     |
| `/settings/connected-accounts` | 连接你的应用，以便你的 Bot 可以与它们协作。                     |
| `/settings/passwords`       | 你在为 Bot 登录网站时保存的登录凭据。                              |
| `/settings/notifications`   | 更新发送到哪里、你的设备，以及最近的投递记录。                     |
| `/settings/approvals`       | 你的 Bot 是否在行动前询问，以及针对每个 Bot 的规则。               |
| `/settings/memory`          | 你的 Bot 关于你记住了什么，以及你告诉它们的事实。                  |
| `/admin/credentials`        | 存储只写加密凭据。                                                 |
| `/admin/approvals`          | 团队审批设置、团队规则，以及共享账户请求。                         |
| `/admin/computers`          | 查看、停止和重置 Bot 计算机。                                      |
| `/admin/boundaries`         | 配置浏览器/文件/MCP 操作策略。                                     |
| `/admin/components`         | 发布组件并管理哪些 Bot 可以使用它们。                              |
| `/admin/playground`         | 在浏览器中起草并发布沙箱化组件。                                   |
| `/admin/plugins`            | 配置 MCP 服务器并将其工具授予 Bot。                                |
| `/admin/skills`             | 编写部署技能并将其授予 Bot。                                       |
| `/admin/learning`           | 为 Bot 分配 Learning 容器，或暂停自动学习。                        |
| `/admin/people`             | 列出、提升、降级和移除已登录的人员。                               |
| `/admin/identity-providers` | 注册公司 SAML 或 OIDC 提供方，按电子邮件域名路由。                 |
| `/admin/enterprise`         | 成员可以使用什么、Bot 计算机如何访问网络，以及记录了哪些相关信息。 |
| `/admin/audit`              | 审查被允许、被拒绝和失败的操作。                                   |

## 功能

- **每个 Bot 一台计算机**：supervisor 为每个 Bot 提供自己的容器、自己的 `/workspace` 卷和自己的浏览器配置文件。在主机支持的情况下，设置 `COMPUTER_RUNTIME=runsc` 可在 gVisor 下运行它们。
- **一个 shell，而不仅仅是浏览器**：Bot 可以在其工作区中运行命令、安装所需内容，并处理它保存的文件。通过与其它一切相同的关卡，因此规则可以完全拒绝 shell 或拒绝特定命令，无论哪种情况该命令都会被记录在案。该命令继承 PATH、locale、终端和代理变量，但不继承部署环境的其余部分。
- **网关是唯一的入口**：它从服务器持有的快照中解析目标、评估策略、写入审计行，然后才调用计算机。不存在先执行操作而没有记录存在的路径。
- **CEL 策略，失败即关闭**：规则可以检查 `tool.name`、`intent`、`bot.id`、`actor.id`、`page.url`、`page.host`、`element.*`、`key`、`command`、`file.*`、`mcp.*` 和 `initiator.*`（是什么启动了这次运行，因此规则可以拒绝定时例行任务做它会允许人做的事情）。拒绝在允许之前评估，缺少策略则不许可任何操作，规则出错时拒绝而非放行。
- **观看它正在做什么**：屏幕显示 Bot 正在看什么，旁边的 Activity 标签显示它运行、读取和保存了什么，以及输出。对话记录中的命令行可以展开查看相同内容。保存的文件显示其路径和大小，从不显示其内容。
- **接管控制**：遇到登录墙或 2FA 提示的 Bot 会请求帮助。控制权在同一面板中移交，并记录为 `computer.help_requested`、`computer.control_taken` 和 `computer.control_released`。当有人驾驶时，Bot 的操作会被拒绝而非排队。
- **机密从不进入对话记录**：记录会记载请求了某个机密以及它的长度，而非它的内容。
- **自带智能体**：任何 AG-UI 端点都是一个 Bot，无论基于框架还是手写。端点使用与浏览器导航相同的目标检查进行验证，认证头以只写方式存储。
- **组件而非文字**：编译后的 React 组件位于 `app/src/components/gallery/`，沙箱化的组件在 `/admin/playground` 中编写并发布，无需部署。每次调用都会向服务器询问该组件是否存在、是否已发布，以及是否对该 Bot 保留。数据函数按组件授予。
- **受治理的 MCP**：Google Drive、Notion 和 [Parallel Search](docs/parallel-research.md) 随目录提供，Composio 通过一个账户代理数百个更多应用，每个都以提问者的身份访问。目录只包含此部署所支持的