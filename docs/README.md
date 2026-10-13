# OpenBot 文档

请从根目录的 [README](../README.md) 开始，然后参考以下文档：

- [架构](architecture.md)：服务、端口、浏览器治理、计算机、组件、插件、知识以及安全边界。
- [配置](configuration.md)：环境变量和租户包 YAML。
- [开发](development.md)：本地环境搭建、迁移、端口和质量检查。
- [协作者](coworkers.md)：持久化的 Bot 配置文件、渠道、可见性、删除以及外部 AG-UI 注册。
- [例程](routines.md)：Bot 按计划运行的常驻指令、触发这些指令的 worker，以及它们以谁的身份运行。
- [自动学习](automatic-learning.md)：哪些 Bot 会向 Learning 容器贡献对话，并接收其发布的技能。
- [并行研究](parallel-research.md)：通过 Parallel Search 连接器进行公开网络搜索和提取。
- 插件，每个连接器一页——管理员注册什么、每个人同意什么，以及各种失败意味着什么：
  - [Composio](plugins/composio.md)：代理服务，因此这是这里唯一一个作为应用目录而非单一连接器的页面。
  - [Google Drive](plugins/google-drive.md)
  - [Notion](plugins/notion.md)
- [部署](deployment.md)：容器、镜像中包含的内容、最小规格以及平台注意事项。
- [Kubernetes](../charts/openbot/README.md)：Helm chart、集群在此之前需要什么，以及各云平台不同的 values 配置。
- [发布](releasing.md)：发布版本如何被提议、审查和发布。
- [Windows 桌面签名](windows-signing.md)：受保护的 Azure Key Vault 签名以及应用和 NSIS 安装程序的验证。
- [桌面提供商 OAuth](../desktop/PROVIDER_OAUTH.md)：发行商在桌面构建中提供 Google 和 xAI 登录所需配置的内容。
- [桌面遥测](../desktop/TELEMETRY.md)：桌面应用发送的每个事件，以及它携带的内容。

请勿在公开文档中包含凭据值、客户数据、对话记录或仅限本地的备注。