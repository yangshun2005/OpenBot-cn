# 协作伙伴

协作伙伴是一个拥有持久化档案和固定角色的 Bot。每次运行时都会发送该角色，因此用户无需在每个频道中重复说明工作内容。

## 数据模型

| 组成部分         | 表                              | 用途                                                                                     |
| ---------------- | ------------------------------- | ---------------------------------------------------------------------------------------- |
| 运行时 Agent     | `agents`                        | 类型（内置、远程 AG-UI 或远程 Mastra）、端点或提示词、可选的密钥引用。                    |
| 档案             | `agent_profiles`                | 名称、头衔、角色、头像种子、所有者、可见性和软删除。                                      |
| 个人名册         | `agent_preferences`             | 每个用户的隐藏和固定状态。                                                                |
| 团队发布         | `team_bot_publications`         | Bot 所有者已发布给整个团队或指定人员和群组的记录。                                        |
| 频道             | `channels`                      | 会话成员关系和协作伙伴绑定。                                                              |
| 智能映射         | `intelligence_channel_mappings` | 频道到线程的映射。                                                                        |

包提供的 Agent 是公开的且无所有者。用户创建的协作伙伴归创建者所有。

## 固定角色

远程协作伙伴会收到一条由其头衔和角色描述生成的系统消息：

```text
You are Expense Manager, Finance Operations.

Review receipts, categorize expenses, and prepare reimbursement reports.

This standing role applies in every channel. Treat channel messages as task-specific instructions within it.
```

部署层会追加一个额外的来源说明块，而非由包提供：它告诉协作伙伴说明每个答案的来源，明确标注哪些是凭自身知识而非来源回答的，并且绝不将后者当作前者呈现。第二个块告诉它如何解读工具结果所携带的不可信内容信封。由于是部署级别的，这两个块不会因为有人添加下一个协作伙伴而被遗忘。

该消息是普通的 AG-UI 系统内容，因此可与任何兼容 AG-UI 的后端配合使用。编辑角色会影响下一次运行。

## 可见性

| 可见性     | 谁可以查看和运行            |
| ---------- | --------------------------- |
| `private`  | 所有者和管理员。            |
| `public`   | 部署中的所有人。            |

过滤在服务端/数据库查询中进行。包提供的 Agent 无法通过产品进行编辑或删除。

所有者还可以将 Bot 发布为团队 Bot，面向所有已登录用户或指定人员和群组（`/api/team-bots`，Bot 的 **共享** 页面）。发布是一条独立的、可撤销的记录；Bot 自身的可见性不会因此改变。已发布的 Bot 在拥有角色描述和非占位符名称（如"New Bot"）之前，对团队成员不可见。

团队 Bot 在所有者已连接它时，通过其**所有者自己的账户**访问个人应用——对发布范围内的所有人（包括触发器）均如此——否则通过提问者自己的账户访问，并需其同意。共享应用是受治理的替代方案：一个属于部署的账户，由管理员审批谁可以通过每个 Bot 使用它。参见 [Composio → 共享应用](plugins/composio.md#shared-apps)。

## 频道

启动频道会创建一个新的会话和 Intelligence 线程。两个使用相同协作伙伴的频道保持独立。

每个频道通过一个频道本地的代理 Agent ID 进行路由，该 ID 固定到该频道的线程 ID，然后转发到协作伙伴的运行时 ID。

## 删除和隐藏

删除是软删除。协作伙伴停止运行，但现有频道对其成员仍可读，并作为墓碑恢复。

隐藏和固定是个人名册状态。隐藏会将协作伙伴从某个用户的列表中移除，但不会为其他任何人禁用该协作伙伴；固定会将其移动到该用户 `/bots` 顶部的 **已固定** 区域，仅对该用户生效。管理员分配给群组的团队 Bot 对其成员始终固定。

## 默认端点

产品创建的协作伙伴使用：

```dotenv
MANAGED_AGENT_AG_UI_URL=http://localhost:4201/ag-ui
```

即 `agent-langgraph`，它运行真实的框架及其自己的工具循环。`4200` 上的概念验证手写了协议，并将循环交给任何正在监听的对象，因此它是参考实现，而非用于构建部署的基础。

该 URL 是可选的。使用 `MANAGED_AGENT_TOKEN` 设置它，或者不设置：产品创建的协作伙伴随后需要自己的端点，而端点展开为空的包 Agent 会被省略，而不是注册到不存在的主机上。没有 URL 的多余令牌会被忽略。包提供的 Agent 在其他情况下使用其自己的 `agents.yaml` 配置。

## 注册外部 AG-UI Agent

在 `agents.yaml` 中：

```yaml
agents:
  - id: risk
    name: Risk
    title: Risk & Compliance
    role_description: Investigate policies and controls.
    type: remote-ag-ui
    endpoint: http://risk.internal/ag-ui
```

在产品中，从 `/bots` 创建协作伙伴，并从其页面的 **设置** 中编辑它，然后设置：

- 名称；
- 头衔；
- 角色描述；
- 可见性；
- 可选端点；
- 可选授权头。

端点注册使用目标检查。在所有配置下都会拒绝云元数据地址。可选密钥是只写的：发送密钥会存储/替换它，省略则保留现有密钥，API 不会返回它。

`POST /api/agents/test-connection` 在保存端点之前检查其是否响应。

## 能力

协作伙伴的角色不授予能力。能力单独治理：

- 浏览器和文件操作通过计算机网关策略；
- 组件在部署范围内发布，可按 Bot 扣留；
- MCP 工具由管理员按 Bot 授予；
- 个人技能只能附加到作者拥有的 Bot 上；
- 部署技能由管理员管理。

参见 [architecture.md](architecture.md)。