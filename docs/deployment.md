# 部署

OpenBot 以单个容器形式发布。它包含应用、为其提供服务的 API，以及 Bot 所驱动的浏览器，还可以自带 PostgreSQL。它在笔记本电脑上能做什么，在部署中就能做什么。

```sh
# 每个发布版本都会发布此镜像，因此部署无需克隆也无需构建。
# `latest` 是最新版本；像 `:v0.0.9` 这样的版本标签则固定到某个版本。
image=ghcr.io/copilotkit/openbot:latest

# 使用你已有的数据库。
docker run -p 3001:3001 --env-file .env "$image"

# 或者使用容器内的数据库。无需其他任何配置。
docker run -p 3001:3001 --env-file .env \
  -e EMBEDDED_POSTGRES=on -v openbot-data:/var/lib/postgresql "$image"
```

从克隆的仓库执行 `docker build -t openbot .` 会生成同样的东西，适合任何部署自己代码树的人。发布版本所发布的内容会在其 `container-images.json` 中固定摘要（digest），部署这些摘要而非移动标签正是 [releasing.md](releasing.md) 所推荐的。

## 镜像中包含什么，不包含什么

**包含：** 构建好的应用、API 和 Chromium。一个端口，3001。浏览器在容器内监听 4100，且刻意不对外发布：它持有真实的登录凭据，唯一的调用者就是它旁边的进程。

**PostgreSQL，如果你需要的话。** `EMBEDDED_POSTGRES=on` 会在容器内启动一个 PostgreSQL，首次运行时创建数据库和 `vector` 扩展，并在每次启动时运行迁移。它仅监听回环地址且从不对外发布，因此无需管理密码。

给它挂载一个位于 `/var/lib/postgresql` 的卷——是父目录，而不是数据目录本身。没有卷的话，重新部署会连同审计追踪一起丢失，而审计追踪正是产品本身。

**挂载父目录，而不是 `/var/lib/postgresql/data`。** 在任何卷为 ext4 挂载的平台（大多数平台都是如此）上，挂载点会带有一个 `lost+found` 目录。`initdb` 不会向一个非空目录中初始化，因此直接挂载到数据目录会导致集群未被创建——而且由于 `api` 会等待 `postgres` 和 `migrate`，容器会启动，平台会报告部署成功，而 URL 却返回 502。挂载父目录则让 `data` 保持为一个普通子目录，这正是 PostgreSQL 所要求的。Docker 命名卷两种方式都可以；平台卷则不行。不提供持久卷的平台应改为指向托管数据库：设置 `DATABASE_URL` 并关闭 `EMBEDDED_POSTGRES`。那里必须启用 `vector` 扩展；RDS、Cloud SQL 和 Azure Database 都支持它，但没有一个会替你启用。

**不包含：**

**supervisor。** 它为每个 Bot 提供自己的容器，这需要 Docker socket，而任何无服务器容器平台都不允许。没有它，每个 Bot 共享同一个浏览器，就像在未配置 supervisor 的笔记本电脑上一样。共享浏览器意味着 Bot 之间共享登录、共享文件和共享会话，对于一个团队信任自己 Bot 的部署来说没问题，但作为租户之间的边界则不行。

**routines 调度。** 此镜像中没有任何东西被调度来触发 routine——API 旁边没有 worker 服务，`worker/`（循环的本地变体）也不在镜像中。但扫描本身在：从 `/app/server` 执行 `bun scripts/fire-routines.ts`，运行一次后退出，这正是 Helm chart 的 CronJob 从同一镜像运行的内容。因此单容器部署需要容器外部的某个东西按计划运行它——外部 cron、平台计划任务，或者以 `--entrypoint sh` 启动的此镜像的第二个容器（不这样做的话，命令会作为 `CMD` 到达，而此镜像的 entrypoint 会在运行它之前先启动一整个第二服务器；参见[迁移](#migrations)）——并且要在**此服务器完整环境之上**设置 `SERVER_INTERNAL_URL` 和 `WORKER_SHARED_SECRET`，而不是替代它。该扫描在查找到期 routine 之前会构建与 API 服务器相同的配置，因此它和服务器一样，缺少加密密钥、Intelligence 值和身份提供者时会拒绝启动：给它相同的 env 文件并加上那两个变量。在有人这样做之前，routine 会被存储、其下次运行时间会被计算、Routines 页面会显示它，但它永远不会触发。参见 [routines.md](routines.md)。

**暂存附件扫描。** 与 routines 调度形状相同，但有一个值得单独说明的后果：拖入编辑器的文件在发送之前就被存储，而此镜像中没有任何东西回收那些从未发送的。扫描是 `bun scripts/cull-staged-attachments.ts`，从 `/app/server` 执行，运行一次后退出，Helm chart 每小时运行一次，删除超过 24 小时的未发送附件——这个时间窗口是脚本的唯一位置参数，因此 `bun scripts/cull-staged-attachments.ts 72` 会保留三天，小数则表示小时的小数部分。它只需要 `DATABASE_URL`——不需要加密密钥、不需要身份提供者、不需要此镜像配置的任何其他东西——因此与上面的 routines 扫描不同，外部 cron 只需设置一个变量即可运行它。此镜像的第二个容器仍然需要 `--entrypoint sh`，原因见[迁移](#migrations)。

在有人这样做之前，被遗弃的上传会累积在 `attachments` 中，直到达到一个人的上限：**每人 32 个未发送文件**，跨所有频道和所有编辑器会话同时计数，并在上传端点被拒绝。一个文件最多 8 MiB，因此每个上传的人最多有 256 MiB 的暂存 blob，这就是存储容量规划所依据的数字。这**不是**编辑器拒绝第九个文件的那个八文件上限：那个上限是在客户端自己请求中命名的桶上计数的，因此它只约束配合的客户端，别的什么也约束不了，这正是其后添加每人上限的原因。

这个上限也是为什么从不运行此扫描比增长更糟糕。一个人在暂存第 33 个文件时看到的拒绝会告诉他们，任何仍未发送的内容都会在一天内被清除——这是代表此扫描做出的承诺。如果没有任何东西运行它，文件永远不会被清除，任何达到 32 的人将永远无法附加任何东西，在任何频道中都是如此。

## 最小规格

在真实镜像上测量，一个 Bot，arm64。

| | 实测 | 最低 | 推荐 |
| --- | --- | --- | --- |
| 内存 | 空闲 409 MB，三次页面加载后 498 MB，快照后 548 MB | **2 GB** | **4 GB** |
| vCPU | 静置时 3% 到 6%，页面渲染时突发 | **1** | **2** |
| 磁盘 | 1.4 GB 镜像 | **4 GB** | 8 GB，为 `/workspace` 留出空间 |

**为什么实测 550 MB 却要 2 GB。** 那个数字是一个 Bot 打开一个页面。每个额外的并发页面大约再增加 100 到 200 MB，而 Playwright 自己的指导是每个并发浏览器预留约 1 GB。2 GB 是一个人使用时不会遇到 OOM killer 的下限；4 GB 是几个 Bot 同时工作时保持舒适的水平。

**不要配置共享内存。** Chromium 以 `--disable-dev-shm-usage` 启动，因此它写入 `/tmp` 而非 `/dev/shm`，64 MB 的默认值无关紧要。这一点很重要，因为 **AWS Fargate 完全不支持 `sharedMemorySize`**；没有那个标志，Chromium 在那里会崩溃，而修复方案将不可用。

## 必需配置

| 变量 | |
| --- | --- |
| `DATABASE_URL` | 带 `vector` 扩展的 PostgreSQL。使用 `EMBEDDED_POSTGRES=on` 时不需要 |
| 身份提供者 | `GOOGLE_OAUTH_*`、`MICROSOFT_OAUTH_*` 或 `OKTA_OAUTH_*`，配合 `BETTER_AUTH_URL`、`BETTER_AUTH_SECRET` 和 `INITIAL_ADMIN_EMAILS`。参见 README |
| `EMBEDDED_POSTGRES` | `on` 表示在容器内运行数据库。默认关闭 |
| `KEY_ENCRYPTION_KEY` | base64 32 字节。`openssl rand -base64 32`。示例密钥在生产环境中会被拒绝 |
| `INTELLIGENCE_API_URL`、`INTELLIGENCE_GATEWAY_WS_URL`、`INTELLIGENCE_API_KEY` | CopilotKit Intelligence。有免费计划，也可以自托管 |
| `COPILOTKIT_LICENSE_TOKEN` | 可选。托管 Intelligence 不签发；为自托管 Intelligence 设置它，其自身部署也需要相同的许可证 |
| 模型密钥 | `OPENAI_API_KEY`，或你配置的提供者 |

如果你不设置 `COMPUTER_TOKEN`，它会在启动时生成。需要它的两个进程都在容器内，因此无需与任何东西共享它。

这里不需要 `MANAGED_AGENT_AG_UI_URL`。镜像不包含 `agent-langgraph` 或 `agent-bot`。保持未设置，随附的 Risk Analyst 同事会被省略，而不是注册到一个不存在的宿主上。仅当 Bot 确实可从此容器访问时才设置它，并配合 `MANAGED_AGENT_TOKEN`。如果你的 `.env` 仍保留笔记本电脑默认的 `http://localhost:4201/ag-ui`，请取消设置它。

**身份验证是必需的。** 未配置身份提供者时，部署会拒绝启动，因为一个每个访问者都是管理员的公共 URL 会静默失败：它看起来像是能工作。配置 Google、Microsoft 或 Okta，或设置 `OPENBOT_SINGLE_USER=true` 来表示你确实想要一个开放部署。`NODE_ENV` 不影响这一点。

该标志不覆盖公共地址。`OPENBOT_SINGLE_USER=true` 与公共互联网可路由到的 `OPENBOT_PUBLIC_URL`、`OPENBOT_APP_URL` 或 `TRUSTED_ORIGINS` 条目同时存在时，现在会拒绝启动并指出该地址，因为公共地址上的开放部署意味着每个访问者都是管理员。私有地址是另一回事，仍然可以运行：家庭 LAN、Tailscale 或 VPN 地址，或 `.local` 名称是允许的，启动时会警告该网络上的任何人都是那个管理员。如果你想在公共地址上开放它，请在前面放置一个登录提供者。

**在前面放置 TLS。** 不仅仅是为了 cookie。从 `http://<address>` 提供的页面不是安全上下文，这会移除一组在 `http://localhost` 上存在、因此在笔记本电脑上从不缺失的浏览器 API。应用不依赖其中任何一个，但登录 cookie 仍然需要 `Secure`，而下面每个平台都会为你终止 TLS。

## 迁移

使用 `EMBEDDED_POSTGRES=on` 时，它们在启动时运行，无需做任何事。只有一个进程且没有部署流水线，因此替代方案将是一份操作手册。

使用外部数据库时，它们是发布步骤，而非启动步骤。两个副本同时启动会竞争，而失败的迁移应停止部署，而不是留下一个半迁移的数据库继续服务流量。

```sh
docker run --rm --env-file .env --entrypoint sh openbot \
  -c "cd /app/server && bun scripts/migrate.ts"
```

**`--entrypoint sh`，它是该命令的关键部分。** 此镜像的 entrypoint 是 `/init`，即 s6 的，镜像名之后的任何内容都是 `CMD`——s6 会在启动镜像中所有内容*之后*运行它。没有这个覆盖，`docker run … openbot sh -c "… migrate.ts"` 会针对你尚未迁移的数据库启动 API 和 Chromium，然后才迁移它：在未迁移的 schema 上运行第二个服务器，这正是本节存在所要避免的竞争，却出现在本应避免它的那一个命令中。替换 entrypoint 则只运行迁移，别的什么都不做。Helm chart 的迁移 Job 是用 Kubernetes 术语表达的同一件事——它设置 `command:`，这会覆盖镜像的 entrypoint 而非追加到它——这就是为什么那条路径从没错过，而这条曾经错过。

用 `scripts/migrate.ts`，而不是 `drizzle-kit migrate`。CLI 是开发依赖，而此镜像用 `bun install --production` 构建，因此它不在里面；它还需要 esbuild 来读取其 TypeScript 配置。在这里要求它迁移时，它会以退出码 1 退出而不说明原因，部署会针对空数据库启动。该脚本使用 `drizzle-orm` 内部的迁移器，这是一个运行时依赖，并保持相同的日志，因此由任一方式迁移的数据库都算已迁移。此镜像自身的启动路径和 Helm chart 的迁移 Job 运行的都是它。

## 副本

Bot 解析元素引用所依据的页面快照存储在 Postgres 中，因此第二个副本可以响应第一个副本快照的点击。如果平台需要，可以运行多个。supervisor 仍然不在此镜像中，因此每个副本共享其中的同一个浏览器。

## 平台说明

**Google Cloud Run。** 将内存设置为至少 2 GB。多于一个实例没问题（参见上面的副本）；每个实例有自己的浏览器，因此 Bot 的登录会留在为其提供服务的那个实例上。Cloud Run 在 gVisor 下运行每个容器，Chromium 对此敏感；在信任它之前先测试一次导航。`gcloud run compose up` 也会部署整个 compose 文件，如果你想同时拥有一个一次性数据库的话。

**AWS。** ECS Express Mode 从 ECR 中的镜像配置集群、负载均衡器、HTTPS 和自动扩缩，是 AWS 现在引导 App Runner 用户转向的方案，因为 App Runner 不再接受新客户。如果你想要任务定义和细粒度 IAM，ALB 后面的 Fargate 上的普通 ECS 是答案。不需要也不可能配置共享内存。

**Kubernetes。** 上面描述的都是手动运行的一个容器。集群是另一种形态，也是唯一一种给 Bot 自己的计算机、无需容器外部的东西即可运行 routines 调度、并将 API 扩展到单个副本之外的形态。那就是 Helm chart：[charts/openbot/README.md](../charts/openbot/README.md)，它用相同的模板覆盖 EKS、GKE、AKS 和普通自托管集群。

**Azure Container Apps。** 带 TLS 和自定义域的托管入口。注意 **240 秒请求超时**：实时屏幕保持长连接，因此预计它会重新连接。基础层每实例并发 WebSocket 上限为 350。

**Railway、Render、Fly.io。** 都直接运行此镜像，且都能一键配置 PostgreSQL，这使它们成为从零到运行部署的最短路径。

## 已知成本

**浏览器镜像只包含 Chromium 浏览器家族。** 一体化 Dockerfile 和发布的 `agent-computer` Dockerfile 都从 Ubuntu 构建，并运行 Playwright 固定的 `install --with-deps chromium` 路径，因此 Firefox 和 WebKit 从未进入最终镜像层。保持该 Playwright 版本与 `agent-computer/package.json` 匹配；只改一个而不改另一个可能导致浏览器协议和可执行文件修订版出现分歧。镜像保留了官方 Node 24.18.1 镜像中的基线 Node 命令行工具（`node`、`npm` 和 `npx`）。以本地 zstd OCI 层描述符对照之前的 Playwright-base `agent-computer` 镜像测量，压缩后的镜像在 arm64 上从 884.2 MiB 降至 535.0 MiB，在 amd64 上从 894.6 MiB 降至 518.1 MiB。

**严格的内容安全策略需要哈希或 nonce。** `app/index.html` 运行一小段内联脚本，在首次绘制前决定主题。此仓库中没有任何东西发送 CSP 头，因此按发布状态它就能工作；在其代理处添加 CSP 头的部署必须显式允许该脚本，否则 `script-src` 会阻止它，页面会在应用启动前以错误的主题渲染。脚本正文的 `'sha256-'` 哈希是无需每请求 nonce 即可在重新构建后存活的版本。