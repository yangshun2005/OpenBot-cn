# 使用 Parallel 进行公共网络研究

Parallel 为 OpenBot 提供公共网络搜索和提取功能。当其 `web_search` 和 `web_fetch` 工具均被授予某个 Bot 时，内置的 Bot 指导会向该 Bot 描述这些工具以用于公共网络研究，同时还会描述它所持有的其他工具。随附的 Research Desk 示例也包含一项公共网络研究技能。此配置以常规授权为前提条件：任何现有 Bot 都不会自动获得新访问权限。

## 启用研究功能

1. 以管理员身份打开 Plugins，从目录中添加 **Parallel Search**。这会固定 `https://search.parallel.ai/mcp` 并发现其工具。匿名轻度使用无需 API 密钥。
2. 将 **web_search** 和 **web_fetch** 授予应研究公共网络的 Bot。对于随附的 fintech 包，选择 **Research Desk**。其 `research-public-web` 技能已声明这些工具；仅声明并不会授予访问权限。
3. 要求 Bot 研究某个主题。它应当进行搜索、阅读选定的来源并引用其 URL。如果你的部署中有许多工具，请运行 `/research-public-web` 以显式选择该示例技能。

对于生产环境或更高限额，请改为添加 **Parallel Search (API key)**，并通过现有的部署凭据流程存储你的 Parallel API 密钥。授予其两个工具并移除匿名授权。两者使用相同的官方端点；经过身份验证的入口会将密钥作为 Bearer token 发送。如果两者均被授予，指导会选择经过身份验证的连接器。密钥保留在服务器端凭据保险库中。

## 选择与控制

管理员可以撤销任一工具、移除连接器，或授予其他提供商。用户可以显式请求使用其他已授权的提供商。现有的工具选择、操作策略、审计记录、供应商错误处理和后台运行授权均保持不变。每次调用都经过 `PluginStore.callTool`；不会向 agent 配置中添加原始 MCP 服务器。

提供商会接收模型选择的目标、搜索查询、请求的公共 URL 以及对话会话标识符。这些参数可能包含来自用户请求的信息：避免发送公共研究不需要的私有上下文。完整对话不会自动转发。免费匿名访问有提供商管理的限制；失败仍然可见，且不会通过其他提供商静默重试。来源 URL 和摘录使用现有的工具结果渲染；此变更不会添加自定义来源卡片 UI。

有关这两个工具、免费访问、API 密钥身份验证和限制，请参阅 [Parallel 官方 Search MCP 文档](https://docs.parallel.ai/integrations/mcp/search-mcp)。