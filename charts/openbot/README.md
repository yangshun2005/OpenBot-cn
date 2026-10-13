# Kubernetes 上的 OpenBot

在任何 Kubernetes 集群上运行 OpenBot：EKS、GKE、AKS 或你自己的集群。一个 Chart，五个目标，它们之间唯一的区别就是 values。

## 集群首先需要什么

本 Chart 假定存在且不会创建的四样东西。

**集群能够拉取的镜像。** 发布版本会公开发布 `ghcr.io/copilotkit/openbot:vX.Y.Z`，而 `image.tag` 需要的正是这个标签。它**仅为 `linux/amd64` 构建**，因此 arm64 节点组（EKS 上的 Graviton、GKE 上的 Tau T2A、AKS 上的 Ampere）无法运行它：Pod 会停留在 `ImagePullBackOff`，这与标签错误或缺少拉取密钥的表现完全相同，所以节点池架构不对是所有人最后才会检查的事情。要么运行 amd64 节点，要么为你拥有的架构构建镜像并推送到集群可以访问的地方。在假设之前先检查：

```sh
docker manifest inspect ghcr.io/copilotkit/openbot:v0.0.9 | grep architecture
```

**Intelligence 凭据。** OpenBot 需要 CopilotKit Intelligence，如果没有 `secrets.intelligenceApiKey`，Chart 会拒绝安装。它来自 CLI，在任何有浏览器的机器上：

```sh
npx --yes copilotkit@latest login           # 浏览器登录
npx --yes copilotkit@latest project select  # 打印 cpk-... 运行时密钥
```

该密钥是托管安装所需的唯一 Intelligence 凭据；免费计划就足够了。`secrets.licenseToken` 是可选的，用于自托管 Intelligence 且自行颁发许可证的场景。`npx --yes copilotkit@latest license --print` 会打印一个许可证而不写入本地 `.env`，这正是你即将粘贴到 Secret 中时想要的方式。使用 `externalSecrets` 时，在其 `data` 中命名为 `license-token`，服务器会以相同方式接收它。

**自托管 Intelligence 也需要该许可证，在其自己的部署上。** 它的 Chart 将其作为 `appApi.licenseSecret` 或 `externalSecrets.secrets.license` 接收。没有它，Intelligence 会对每次权限读取返回 `RUNTIME_ENTITLEMENTS_SELF_HOSTED_MISCONFIGURED` 并拒绝记忆，而对话仍会继续；OpenBot 在启动时会记录 `intelligence-entitlement-not-ready`。

**一个默认 StorageClass**，或一个具名的 StorageClass。Bot 的计算机和捆绑的数据库都需要卷，而新集群通常没有标记为默认的 StorageClass。参见[首先检查默认 StorageClass](#首先检查默认-storageclass)，这是首次安装时 Pod 卡在 `Pending` 且没有任何提示的最常见原因。

**一个数据库，以及命名它的 Secret**，除非你使用捆绑的那个。Chart 从你创建的 Secret 中读取 URL；它绝不会将你的数据库凭据写入 values 文件：

```sh
kubectl create namespace openbot
kubectl -n openbot create secret generic openbot-database \
  --from-literal=database-url='postgresql://USER:PASSWORD@HOST:5432/openbot?sslmode=require'
```

然后 `--set database.existingSecret=openbot-database`。键必须是 `database-url`，或者用 `database.existingSecretKey` 指定其他名称。参见[你自己的数据库](#你自己的数据库真实部署所使用的)了解 `sslmode` 和 `vector` 扩展，托管数据库在这两方面都会以指向错误问题的方式失败。

### 从零开始搭建 EKS 集群

上述三样东西，作为一个配置和两条命令。`eksctl` 会创建 `gp2` 且不将其标记为默认，并且它指定的 provisioner 是当前 Kubernetes 已不再拥有的树内 provisioner，因此下面的 StorageClass 不是可选的。

```yaml
# cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata: { name: openbot, region: us-east-2, version: "1.34" }
iam: { withOIDC: true }
addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
  - name: metrics-server
  # 最后创建，因为它需要 OIDC provider，而 OIDC provider 需要控制平面。让它
  # 完成；在此运行期间手动创建相同的 addon 会导致集群创建失败。
  - name: aws-ebs-csi-driver
    wellKnownPolicies: { ebsCSIController: true }
managedNodeGroups:
  - name: workers
    # amd64：已发布的镜像没有 arm64 变体。参见上面的镜像说明。
    instanceType: t3.large
    desiredCapacity: 2
    minSize: 2
    maxSize: 4
    volumeSize: 60
    volumeType: gp3
    # Bot 有一个 shell，而 shell 可以触及其 Pod 所能触及的一切。这两项将节点
    # 自身的 IAM 角色置于其触及范围之外。参见下文。
    disableIMDSv1: true
    disablePodIMDS: true
```

**为什么需要这两行 IMDS 配置。** 节点的实例元数据服务位于 169.254.169.254，它会将该节点的 IAM 角色交给任何从该节点发起请求的东西。Bot 的计算机是该节点上运行模型所选命令的 Pod，因此针对该地址的 `curl` 只需一次 `run_command`，而 OpenBot 自己的策略引擎中的任何规则都不会被查询：引擎决定浏览器导航和工具调用，而 shell 直接访问网络。

`disablePodIMDS` 将跳数限制设为 1，因此来自 Pod 的数据包（比来自节点本身的数据包多一跳）不再到达。该跳数限制适用于 IMDSv2 令牌 PUT，而 IMDSv1 是没有 PUT 可限制的裸 GET，所以只有在令牌成为强制要求后跳数限制才有意义。它已经是强制要求了：eksctl 文档说明 `disableIMDSv1` 默认为 true，并为任一标志设置 `httpTokens: required`。这里写出 `disableIMDSv1: true` 是为了陈述这一点而非改变它，因为判断此配方是否安全的读者不应该需要知道上游默认值。注意这覆盖集群网络上的 Pod；以 `hostNetwork: true` 运行的 Pod 位于节点上，仍然可以访问该地址。

**跳数限制的代价。** EBS CSI 驱动从 IMDS 读取 EC2 实例元数据，其自己的 FAQ 要求在像 EKS 这样的容器化环境中"将 IMDSv2 响应的跳数限制……设为 2 或更大"。跳数限制为 1 也排除了 `MutableCSINodeAllocatableCount`，其文档化的前提条件是驱动"必须使用 IMDS 元数据"且 IMDS 是其唯一启用或首选的元数据源。如果你的集群需要其中任何一项，请删除这两行，转而依赖下面的 NetworkPolicy。

有两件事 eksctl 会拒绝而不是警告，在下一次编辑此文件之前都值得了解：`disablePodIMDS` 不能与同一节点组上的 `iam.withAddonPolicies` 组合使用，并且这两个字段都不能设置在提供自己 `launchTemplate` 的节点组上。

下面的 NetworkPolicy 也例外放行 `169.254.0.0/16`，在强制执行它的地方是更好的控制手段。它默认关闭，在 EKS 上除非告知 VPC CNI 强制执行它，否则不起作用，因此这两行才是集群实际启动配置中起作用的。OpenBot 中没有任何东西需要 Pod 级别的 IMDS：Chart 通过 IRSA 访问 AWS，这是 ServiceAccount 注解和投影令牌，而非节点的角色。

```sh
eksctl create cluster -f cluster.yaml

kubectl apply -f - <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations: { storageclass.kubernetes.io/is-default-class: "true" }
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters: { type: gp3 }
EOF
```

数据库放在同一个 VPC 的私有子网中，安全组允许来自集群自身安全组的 5432 端口。`aws eks describe-cluster` 会列出两者。保持 `--no-publicly-accessible`：唯一需要访问它的东西就在集群内。

## 安装

捆绑的数据库和一个管理员，这是最短的可用配置：

```sh
helm dependency build charts/openbot
helm upgrade --install openbot charts/openbot \
  --namespace openbot --create-namespace \
  --set postgresql.enabled=true \
  --set config.initialAdminEmails=you@example.com \
  --set-string secrets.keyEncryptionKey="$(openssl rand -base64 32)"
```

`secrets.keyEncryptionKey` 加密凭据保险库。生成一次，保留它，不要把它放进任何人会提交的文件中。Chart 将它创建的 Secret 标记为 `helm.sh/resource-policy: keep`，因此卸载不会带走每个已存储凭据加密所用的密钥。

### 通过摘要固定镜像

标签可以被移动；摘要不能。每个发布版本的 `container-images.json` 列出了 CI 测试过的摘要，`image.digest` 部署的正是它：

```sh
gh release download v0.2.0 --pattern container-images.json
helm upgrade --install openbot charts/openbot ... \
  --set image.tag=v0.2.0 \
  --set-string image.digest="$(jq -r '.images.openbot.reference | split("@")[1]' container-images.json)"
```

镜像变为 `repository:tag@digest`，因此标签保持可读，而摘要决定拉取的内容。任何不是 `sha256:` 加 64 个十六进制字符的内容都会在安装时被拒绝。

### 你自己的租户包

租户包是一个目录（`config.tenantPackageDir`，默认为 `/app/examples/fintech`）。要在不构建镜像的情况下运行你自己的包，将其文件放入 ConfigMap 并用 `config.extraVolumes` 和 `config.extraVolumeMounts` 挂载。它们会到达服务器以及 routines 和 computer-culler CronJob——加载该包的 Pod，因此无论在哪里读取，包都是相同的：

```yaml
config:
  tenantPackageDir: /tenant
  extraVolumes:
    - name: tenant
      configMap: { name: my-tenant-package }
  extraVolumeMounts:
    - name: tenant
      mountPath: /tenant
      readOnly: true
```

ConfigMap 是扁平的，因此使用 `agents/` 的包需要 `items` 配合 `path: agents/<file>.yaml`，或者将其协作者放在 `agents.yaml` 中。

## 默认值假定什么

**一个没有云功能的普通集群。** 集群自己的默认 StorageClass、没有 RuntimeClass、一个普通的 Kubernetes Secret、一个 Ingress。模板中没有任何云分支，也永远不应该有。托管集群上的部署会开启一些东西；自托管的部署什么都不改，仍然可以工作。

默认两个副本，因为水平扩展才是重点。所有必须在副本间存活的东西都在 PostgreSQL 中，而一个副本会隐藏所有不是的 bug。

**API Pod 中没有浏览器。** 镜像在 API 旁边运行 Bot 的计算机，以便一个容器可以独立工作。副本不应携带一个：浏览器是几百兆字节，持有某个 Bot 的登录信息，因此扩展 API 会随之扩展这些。这里 `server.embeddedComputer` 是关闭的，在超过一个副本的情况下请求它会在安装时被拒绝。

## 你自己的数据库，真实部署所使用的

```sh
--set postgresql.enabled=false \
--set database.existingSecret=openbot-database   # 键：database-url
```

`postgresql.enabled` **默认关闭且非生产级**。Pod 上的数据库会随 Pod 一起消失：一次 rollout、一次节点排空或一次驱逐就是一次重启，虽然卷会存活，但这种形态不会给你备份、故障转移或时间点恢复。它的存在是为了让某人可以用一条命令试用 OpenBot。

将其指向 RDS、Cloud SQL、Azure Database 或你自己的服务器，并将 URL 保存在 Secret 中而非 values 文件中。同时设置捆绑数据库和 URL 会被拒绝，而不是让其中一个静默胜出。

**在 URL 上加上 `?sslmode=require`。** 每个托管数据库都拒绝未加密连接：RDS 默认开启 `rds.force_ssl`，Cloud SQL 和 Azure Database 也是如此。没有它，迁移会失败并报 `no pg_hba.conf entry for host ... no encryption`，这指出了主机和用户而非实际问题。

服务器接受 `disable`、`require` 和 `verify-full`（配合 `sslrootcert=/path/to/ca.pem`，或不带它使用系统根证书）。`require` 加密但不检查任何东西，与 libpq 的一样。`verify-full` 需要 Bun 1.4 或更高版本，因为 Bun 1.3 会连接它被展示的任何证书，所以服务器在 1.3 上拒绝它而不是假装检查。`verify-ca`、`prefer` 和 `allow` 被拒绝：Bun 既不能在没有主机名的情况下检查颁发机构，也不能在 TLS 和明文之间回退。

**执行迁移的角色必须 OWN `vector` 扩展，而不仅仅是能看到它。** 第一次迁移运行 `CREATE EXTENSION IF NOT EXISTS vector`，迁移 `0010` 在文档索引消失后运行 `DROP EXTENSION IF EXISTS "vector"`。`DROP EXTENSION` 是所有权检查，而 `IF EXISTS` 不会免除它——它只是让*缺失*的扩展不报错。因此，一个能看到别人拥有的扩展的角色会通过创建但在删除时失败，迁移 Job 会停止并报：

```
must be owner of extension vector
```

在 Postgres 中，扩展的所有者是运行 `CREATE EXTENSION` 的人，并且没有 `ALTER EXTENSION ... OWNER TO` 可以在之后移交。因此，以前这里给出的建议——让管理角色创建一次，以便 `CREATE EXTENSION IF NOT EXISTS` 对普通用户通过——正是产生失败的原因：它让管理员成为所有者，而迁移角色成为旁观者。

改为执行以下之一：

- **让迁移角色创建它。** 最简单，不需要额外步骤：允许该角色运行 `CREATE EXTENSION`（RDS 上的 `GRANT rds_superuser`、Cloud SQL 上的 `cloudsqlsuperuser`、Azure Database 上的 `azure_pg_admin`，加上供应商维护的任何扩展允许列表），然后让扩展保持缺失，由迁移 `0000` 创建它。迁移角色拥有它，`0010` 干净地删除它。
- **以迁移角色身份预创建它。** 在无法获得该授权的地方，管理角色仍然可以代表另一个角色执行，这就是全部诀窍：

  ```sql
  SET ROLE openbot_migrator;      -- DATABASE_URL 中的角色
  CREATE EXTENSION IF NOT EXISTS vector;
  RESET ROLE;
  ```

  所有权跟随运行语句的角色，因此这等同于第一个选项。

如果你已经安装了扩展且它由别人拥有，在升级前以迁移角色删除并重新创建——以所有者身份执行 `DROP EXTENSION vector;`，然后执行上面的代码块。你的东西不在其中：`vector` 是为 `chunks` 上的 `embedding` 列而存在的，`0010` 在同一事务中删除它。添加了自己的向量列的部署是唯一例外，`0010` 被编写为失败而不是拿走它；该部署应保留扩展并仅手动应用表删除。

## 五个目标

`ci/` 为每个目标保存一个 values 文件，每个都是表达该集群不同之处的最短配置：

| 文件 | 展示内容 |
| --- | --- |
| `self-hosted-values.yaml` | 什么都不开启。如果这个文件需要增长，说明某个默认值是错的。 |
| `eks-values.yaml` | IRSA、Secrets Manager、ALB、区域分布、自动扩缩容。 |
| `eks-sandbox-values.yaml` | 相同，但每个 Bot 一个计算机而非一个共享浏览器。`shared` 和 `sandbox` 渲染不同的 Deployment、不同的 RBAC 和不同的 Pod 模板，因此只渲染其中一个的目标只检查了 Chart 的一半。 |
| `gke-values.yaml` | Workload Identity、Secret Manager、Gateway API 而非 Ingress。 |
| `aks-values.yaml` | Workload identity、Key Vault、AKS Web 应用路由类。 |

无需集群即可渲染其中任何一个：

```sh
helm template openbot charts/openbot -f charts/openbot/ci/eks-values.yaml
```

### 身份，一个映射搞定

EKS 上的 IRSA、GKE 上的 Workload Identity 和 AKS 上的 workload identity 都是 ServiceAccount 上的注解，因此 `serviceAccount.annotations` 覆盖全部三种，Chart 无需知道自己在哪个云上。

### 密钥，无需供应商

普通的 Kubernetes Secret 是默认值，因为这是自托管集群所拥有的。设置 `externalSecrets.enabled` 会将相同的键转换为针对集群所拥有的任何存储的 ExternalSecret，因此 Secrets Manager、Secret Manager 和 Key Vault 是一个 values 块而非三条代码路径。

### 首先检查默认 StorageClass

全新的 EKS 集群经常没有默认 StorageClass。`eksctl` 创建 `gp2`，它未被标记为默认，并使用当前 Kubernetes 已不再拥有的树内 `kubernetes.io/aws-ebs` provisioner。请求"默认"的卷因此永远不会绑定，计算机会停留在 `Pending`，没有任何提示说明原因。一行命令告诉你：

```sh
kubectl get sc
```

要么创建一个由 `ebs.csi.aws.com` 支持的默认类，要么将 `computers.persistence.storageClass` 和 `postgresql.primary.persistence.storageClass` 设置为一个存在的类。`ci/eks-values.yaml` 做的是后者。

`volumeBindingMode: WaitForFirstConsumer` 在每个云上都很重要：没有它，卷会在 Pod 被调度之前选择的区域中创建，Pod 会因节点亲和性冲突而无法调度。这只发生在多区域集群中，因此它会通过每个单区域测试。

### 存储有引力

API 层不在磁盘上保存任何东西。当每个 Bot 的计算机到来时它们会保存，而所有三个云上的普通块存储都是**区域性的**：一旦配置，引用它的每个 Pod 都会被调度到该区域，因此 Bot 的计算机在其配置文件存在期间被固定在一个区域。这是可接受的，值得说明而非发现。`storageClass`