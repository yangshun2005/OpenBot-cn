# 开发

## 环境搭建

安装 Docker、[Bun](https://bun.sh) 1.3+、`lsof`、`openssl` 和 `curl`。下面的 Intelligence 配置还需要 `npx`（Node）；`scripts/start.sh` 在首次运行时使用 `openssl` 生成密钥。

```sh
cp .env.example .env
bun install
```

在 `.env` 存在后配置 CopilotKit Intelligence：

```sh
npx --yes copilotkit@latest login
npx --yes copilotkit@latest project select
bun scripts/setup-learning.ts
```

对于全新的托管配置，该辅助脚本会配置所选项目的密钥，并创建或复用其 `openbot` Learning 容器。只有在验证容器之后，它才会将 `INTELLIGENCE_API_KEY` 和 `CPK_INTELLIGENCE_LEARNING_CONTAINER_ID` 写入 `.env`。已有的密钥和自定义目标会被保留。对于全新的自托管配置，将 `INTELLIGENCE_API_URL` 设置为部署的 HTTPS API 源，并仅运行 `bun scripts/setup-learning.ts`。请先安装 Chrome 或 Edge：该辅助脚本会为部署的常规登录打开一个隔离的浏览器窗口，让你选择一个可访问的项目，并在铸造其运行时密钥之前创建或复用 `openbot`。本地开发可以使用回环 HTTP 源。已有的仅密钥配置使用经过身份验证的 Intelligence 管理来创建容器，并在 OpenBot Admin 中分配它。没有托管许可证步骤。然后添加 `OPENAI_API_KEY`。

启动整个技术栈：

```sh
bash scripts/start.sh
```

## 运行服务

使用 `bash scripts/start.sh` 启动完整的本地技术栈。它会启动 Docker 服务、应用迁移、启动 API 服务器、应用和例行 worker，并验证健康检查路由。

使用 `bash scripts/stop.sh` 将其关闭：包括应用、例行 worker、API 服务器、Docker 服务，以及每个 Bot 的 computer——后者由 supervisor 创建而非 compose，因此其生命周期会超过 `docker compose down`。传入 `--keep-computers` 可让这些浏览器保持登录状态。无论哪种方式都不会删除任何内容。

仅当你想要应用和 API 服务器而不启动 Docker Bots 和 computers 时，才使用 `bun run dev`。

| 服务              | 端口                       |
| ----------------- | -------------------------- |
| `app`             | 3010                       |
| `server`          | 3001                       |
| `agent-computer`  | 4100                       |
| `agent-bot`       | 4200                       |
| `agent-langgraph` | 4201                       |
| `supervisor`      | 4500 主机 / 4300 容器      |
| PostgreSQL        | 5432                       |

`start.sh` 每次运行都会将每个选定的 Docker 服务交给 `docker compose up -d`，因此配置发生变化的服务会被重新创建，而未变化的服务则保持原样。它会放过已经作为 OpenBot 响应的 API 服务器或应用，在 API 服务器拒绝 worker 的密钥或没有 routines 路由时重启它，并在端口被其他进程占用时进行报告。

**这里没有任何东西会清理暂存的附件。** 拖入编辑器的文件会在消息发送之前被存储，而唯一能回收那些从未发送的文件的机制是 `server/` 中的 `bun scripts/cull-staged-attachments.ts`，Helm chart 每小时运行一次，而 `docker-compose.yml` 和 `start.sh` 都不会启动它。它只需要 `DATABASE_URL`，并以小时为单位接受一个可选的保留窗口参数，默认为 24。在笔记本电脑上这通常无关紧要，因为这些行很小，而且数据库是你自己的。但到了三十二就不再无关紧要了：一个人可能同时在所有频道中持有那么多未发送的文件，对下一个文件的拒绝会承诺它们会在一天内被清除，而在没有任何清理的地方它们并不会被清除，因此长期运行的本地部署可能会达到无法附加任何文件的状态。在编辑器中删除文件会直接删除它，因此这针对的是被放弃的草稿而非普通使用。[deployment.md](deployment.md) 对真实部署也有同样的说明。

## 迁移

更改 Drizzle schema 之后：

```sh
bun run --filter server db:generate
bun run --filter server db:migrate
```

在共享生成的迁移文件之前请先审查它们。`start.sh` 在启动技术栈时会应用已有的迁移。

**不要手动编辑生成的迁移。** 这会导致文件与生成器产出的内容不再匹配。如果生成的 SQL 无法工作——例如 `ADD COLUMN ... NOT NULL` 在已有行的表上失败——请改为拆分它：先生成可空列，添加数据步骤，然后生成约束。

**收紧现有列的约束属于后续发布版本**，而不属于添加该列的发布版本。滚动部署会运行迁移，然后同时由新旧副本提供服务，而旧副本写入的行不包含新列：在 `NOT NULL` 下，它的写入会开始失败，因此添加该列的发布版本会对所有落在尚未被替换的副本上的用户造成破坏。请以可空方式发布该列，让整个集群完成轮换，然后再收紧它。`accounts` 上的 `issuer` 就是一个实例：该列是可空的，且没有任何迁移收紧它。

**数据步骤是它自己的迁移**，使用为此存在的标志创建：

```sh
bun run --filter server db:generate -- --custom --name=backfill_something
```

生成器是对 schema 与 schema 进行差异比较，因此像"provider 为 Google 的行获得 Google 的 issuer"这样的规则永远不可能由它产生：它不在 schema 中。`--custom` 会写入一个在日志中注册的空文件，这是唯一应该由任何人手写的迁移。

CI 对此强制执行两件事：`drizzle-kit check` 用于检查迁移之间的冲突和间隙，以及一个"生成并在有差异时失败"的探测，它会拒绝没有为其编写迁移的 schema 更改。

**如果 `drizzle-kit migrate` 挂起然后以非零状态退出且没有错误**，说明日志中指定了一个不存在的迁移文件。变基会导致这种情况：`meta/_journal.json` 是一个已检入的文件，因此恢复它可能会重新引入已被重命名的迁移的条目。在这种状态下 `drizzle-kit check` 会报告"Everything's fine"，因为它比较的是 schema，而不是检查日志与目录是否一致。请将 `meta/_journal.json` 与 `ls server/drizzle/*.sql` 进行对比。

## 质量检查

在打开 Pull Request 之前运行这些检查：

```sh
bun run format:check
bun run lint
bun run typecheck
bun run test
bun run build
```

集成测试需要一个带有 pgvector 的 PostgreSQL 数据库。在运行它们之前，将 `TEST_DATABASE_URL` 指向一个专用数据库，例如 `postgres://openbot:openbot@localhost:5432/openbot_test`。

它们拒绝使用应用的 `openbot` 数据库。`DATABASE_URL` 仍然是应用设置，而数据库客户端在打开连接后会将其从进程环境中移除，以保留 Windows Bun 连接修复，因此测试使用 `TEST_DATABASE_URL` 作为其不可变的固定地址。针对你正在使用的部署运行它们会将测试 Bot 放入其审计追踪和活动报告中，因此请创建一个单独的数据库并在运行集成测试套件之前对其进行迁移。

CI 使用 `bun run test:ci` 在常规测试之外验证预期的测试数量。

`bun run test:smoke` 是独立的，需要一个已启动的部署，并在其上有一个会话：

```sh
bash scripts/start.sh
export OPENBOT_SMOKE_COOKIE='better-auth.session_token=...'
bun run test:smoke
```

它通过 HTTP 针对正在运行的技术栈驱动一个完整流程，因此它覆盖了其余测试套件无法触及的连接：服务器到 supervisor 到 computer、网关在浏览器行动之前做出决定，以及审计行的落地。使用 `OPENBOT_API_URL` 将其指向其他地方。没有部署时，它会被 `bun run test` 跳过，并在被点名要求时说明需要启动什么。

该会话不是可选的，也不是为了方便。该流程所验证的每条路由都在 `requireUser` 之后，因此没有会话时，三个在 computer 上操作的测试会返回 401，运行会在没有任何问题的情况下报告部署已损坏。请从已登录到被测部署的浏览器中获取 cookie，位置在 DevTools 的 Application、Cookies 下。它是一个具有该人员权限范围的凭据：它属于运行的环境，而不属于文件或 Pull Request 评论。在没有它的情况下被要求运行时，运行会在第一个测试之前停止并指出该变量名。

`bun run test:live-screen` 出于相关原因是独立的，且不需要部署，只需要此目录自身的依赖：

```sh
cd agent-computer && bun install
cd .. && bun run test:live-screen
```

它使用真实的 Chromium 针对真实的 computer 进程驱动实时屏幕：浏览器仍在启动时套接字关闭、第二个连接从第一个连接接管屏幕、滚轮拒绝来自拥有它的套接字的输入，以及浏览器因请求或空闲清理而关闭。这些需要 `agent-computer/src/index.ts`，它在模块作用域导入 Playwright，而 `playwright` 仅在 `agent-computer/package.json` 中声明，根目录的 `bun install` 无法触及。因此，在没有 `OPENBOT_LIVE_SCREEN=1` 的情况下，这些文件会在导入任何内容之前跳过，这正是让 `bun run test` 和 CI 在从未安装该依赖的地方仍能工作的原因。

## 贡献清单

- 保持更改聚焦。
- 将凭据、服务账户 JSON、客户数据和转录内容排除在源代码管理之外。
- 将敏感行为放在服务器上，而不仅仅放在浏览器中。
- 当行为发生变化时，更新[配置](configuration.md)、[架构](architecture.md)或根目录的 [README](../README.md)。
- 运行上述质量检查，并将结果包含在 Pull Request 中。