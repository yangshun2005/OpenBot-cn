# 发布

一次发布就是一个人选定版本，然后由一个经过审查的 Pull Request 完成其余所有事情。没有任何步骤涉及终端、手动推送的标签，或在某人笔记本电脑上构建的镜像。

## 执行一次发布

1. 检查 [CHANGELOG.md](../CHANGELOG.md) 中的 `## Unreleased` 是否符合你的预期。它就是发布说明。不会从提交主题生成任何内容，因为提交主题是写给阅读 diff 的人看的，而这些说明是写给决定是否升级的人看的。
2. 在 Actions 标签页中运行 **Create release PR**，选择 `patch`、`minor` 或 `major`。如果你想在不打开任何东西的情况下查看版本和说明，可以先使用 `dry_run`。
3. 审查它打开的 Pull Request。它会更新 `package.json`，将 changelog 中的 `## Unreleased` 标题提升为 `## X.Y.Z`，并将 Helm chart 的 `appVersion` 以及桌面端 Cargo 包/锁文件移动到相同的版本。Tauri 直接从根目录的 `package.json` 读取其版本。
4. 合并它。这就是发布。

合并即是触发条件，因此一次发布始终是 `main` 上一个经过审查的提交。

## 桌面端构建版本

根目录的 `package.json` 是发布版本的来源。CI 会拒绝桌面端 Cargo manifest 或锁文件中的偏差。发布工作流会自动更新这些文件；在手动更改根版本之后，运行 `bun desktop/scripts/desktop-version.ts sync`。

内部桌面端构建产物使用 `X.Y.Z-internal.g<commit>`，以便测试人员能够识别源修订版本。桌面端 CI 和受保护的 Windows 签名都会在编译和打包之前准备此版本。构建产物包含 `build-version.json`，其中记录了完整的提交和发布版本。

要在本地构建，请从仓库根目录运行：

```sh
bun desktop/scripts/desktop-version.ts internal
cd desktop
APPLE_SIGNING_IDENTITY=- bun run tauri build --config src-tauri/tauri.build-version.conf.json --bundles dmg
```

使用 `release` 代替 `internal` 来准备普通的发布版本。这只会构建一个构建产物；它不会发布一个发布版本。macOS 保留数字系统版本字段，并将完整的内部标识符存储在应用 `Info.plist` 中的 `OpenBotBuildVersion` 里。Windows 在 `ProductVersion` 和 `FileVersion` 中保留完整标识符；其固定的数字字段包含发布编号。受保护的签名任务会检查这两个嵌入的字符串版本。

临时签名的 Mac 构建需要 [Apple 的说明](https://support.apple.com/en-us/102445)中描述的首次打开例外。它们用于内部测试，未经 Apple 公证。有关签名的 NSIS 构建，请参阅 [Windows 签名](windows-signing.md)。

## 合并会做什么

`publish-release.yml` 在每次推送到 `main` 时运行，并首先判断该提交是否是一次发布。它向 API 查询产生该提交的 Pull Request，并要求 head 分支匹配 `release/publish/vX.Y.Z`、该分支位于此仓库中，并且该 Pull Request 带有 `release` 标签。Fork 可以将分支命名为任何名称；但它无法添加标签。

然后，按顺序：

- 树中的版本会与正在发布它的分支进行核对，并且会检查 changelog 中是否存在具有该编号的章节
- 构建一个镜像并推送到 `ghcr.io/copilotkit/openbot`，打上版本、提交和 `latest` 标签
- `docker-compose.yml` 可以构建的服务也会被发布，每个服务一个镜像，位于 `ghcr.io/copilotkit/openbot-<service>`。这些镜像是 `linux/amd64` 和 `linux/arm64`，在各自架构的原生 runner 上构建，并合并到一个 manifest list 中，因为拉取它们的机器既有笔记本电脑也有服务器。`.github/published-images.json` 就是该列表，如果它与树中的 Dockerfile 不再匹配，CI 就会失败
- 使用工作流的 OIDC 身份签署构建来源证明，并与每一个镜像一起推送
- 为该提交打上标签并创建一个 GitHub Release，将 changelog 章节作为其说明，并将 `container-images.json` 作为附件

## 部署一个发布版本

`container-images.json` 为发布版本发布的每个镜像固定了一个 digest。部署这些，而不是标签：

```sh
gh release download v0.1.0 --pattern container-images.json
docker run -p 3001:3001 --env-file .env \
  "$(jq -r .images.openbot.reference container-images.json)"
```

同一个文件也是机器在不构建任何内容的情况下运行整个技术栈的方式。`images` 下的每个键都是一个服务，因此可以直接从中读取引用：

```sh
export COMPUTER_IMAGE="$(jq -r '.images["agent-computer"].reference' container-images.json)"
export SUPERVISOR_IMAGE="$(jq -r .images.supervisor.reference container-images.json)"
export BOT_IMAGE="$(jq -r '.images["agent-bot"].reference' container-images.json)"
export LANGGRAPH_IMAGE="$(jq -r '.images["agent-langgraph"].reference' container-images.json)"
export SERVER_IMAGE="$(jq -r .images.server.reference container-images.json)"
export IMAGE_PULL_POLICY=missing
docker compose up -d
```

`IMAGE_PULL_POLICY=missing` 是将这些名称转变为拉取的关键；没有它，这些服务中的每一个都会从源码构建，因为带有 `build` 部分的服务无论其镜像叫什么名字，默认都会构建。拉取失败仍会回退到构建，因此没有工具链的机器需要将 `build` 部分覆盖掉，而不仅仅是依赖这个变量。

标签可以移动到指向不同的镜像；digest 则不能。CI 冒烟测试过的同一个 digest 就是运行的那个，而回滚就是用更早的版本执行相同的命令。

在 Kubernetes 上，chart 将相同的 digest 作为 `image.digest`；请参阅 [charts/openbot/README.md](../charts/openbot/README.md#pin-the-image-by-digest)。

在部署之前，你可以检查某个镜像是否是由此仓库构建的。每个已发布的镜像都带有自己的证明：

```sh
gh attestation verify oci://ghcr.io/copilotkit/openbot:v0.1.0 -R CopilotKit/OpenBot
gh attestation verify oci://ghcr.io/copilotkit/openbot-supervisor:v0.1.0 -R CopilotKit/OpenBot
```

## 哪些必须通过

分支保护应要求一项检查，即 `verify`，除非其他所有任务都成功，否则它会失败。添加到 `ci.yml` 的任务会自动被它覆盖，无需任何人更新列表。

| 检查 | 它能捕获什么 |
| --- | --- |
| `format, lint, types` | 常规问题，覆盖每个工作区，包括 `agent-computer` 和 supervisor |
| `types (agent-computer)`、`types (supervisor)` | 两个可部署包中任一包的类型错误，各自在其独立的安装上检查 |
| `native types` | `mobile` 应用中的类型错误，该应用从其自己的 npm 锁文件安装 |
| `computer (real browser)` | 只有真实 Chromium 才能展现的 agent-computer 行为，例如密码进入模型读取的快照、重启时丢失的会话 cookie，或 WebRTC 绕过出口过滤器 |
| `tests` | 孤立情况下做出的错误决策 |
| `chart` | 一个 Helm values 文件渲染出的服务器无法启动，覆盖 self-hosted、EKS、EKS sandbox、GKE 和 AKS 目标 |
| `python harness regressions` | Python Bot harness 中的 provider 边界回归 |
| `startup (macos-latest)`、`startup (windows-latest)` | 应用的 serve-or-build 缓存在 macOS 或 Windows 上做出错误决策 |
| `build` | 应用无法编译 |
| `migrations` | 没有迁移的 schema 变更，或已漂移的快照 |
| `image` | 能构建但无法启动的镜像，或不断重启的受监督服务 |
| `component dockerfiles` | 发布版本会发布的某个 Dockerfile 不再能构建，或发布列表已不再覆盖它 |

`image` 的重要性超过其位置所暗示的。它上面的所有检查都可能在一棵镜像永远无法启动的树上通过，因为这里没有其他东西运行它所交付的东西。它会构建容器，使用嵌入式 PostgreSQL 启动它，等待 `/api/capabilities`，并在受监督服务不断重启时失败。

当发布 PR 被合并时，这些检查会再次针对发布提交运行。它们把关的是发布而非提案，这就是为什么发布 PR 到达时没有自己的检查并不重要：由工作流打开的 Pull Request 不会触发它们。

**发布不需要任何密钥。** CI 和两个发布工作流只使用内置的 `GITHUB_TOKEN`。桌面端构建工作流还会读取 `OPENBOT_GOOGLE_MODEL_OAUTH_CLIENT_SECRET`。

## CI 无法做的一件事

`tests/smoke` 中的冒烟旅程是唯一证明各部分彼此连接的检查：服务器到达 supervisor，supervisor 构建一个 computer，网关在浏览器行动之前做出决策，并且轨迹记录了它。它无法在 CI 中运行，而这不是一个以后要填补的空白。

OpenBot 只以 Intelligence 模式运行，而 `loadConfig` 在没有项目的 Intelligence 值时会拒绝启动，托管 runner 没有理由持有这些值。`image` 检查通过占位值绕过了这一点，因为启动时不会联系任何东西，但该旅程断言 `licenseStatus` 为 `valid`，而没有任何占位值能让它为真。

因此，这是一个人在合并发布 PR 之前，在拥有真实 Intelligence 凭据的机器上执行的一个步骤：

```sh
bash scripts/start.sh
export OPENBOT_SMOKE_COOKIE='better-auth.session_token=...'   # from a signed-in browser
bun run test:smoke
```

会话是"拥有真实凭据的机器"的另一半：该旅程所证明的路由位于 `requireUser` 之后，因此没有会话的运行会三次返回 401，并且对发布版本没有任何说明。有关 cookie 的来源，请参阅 [development.md](development.md#quality-checks)。

发布 PR 会在评论中要求提供结果。这刻意由人而非机器人来完成：它是唯一无法自动化的关卡，因此也是唯一值得点名的关卡。