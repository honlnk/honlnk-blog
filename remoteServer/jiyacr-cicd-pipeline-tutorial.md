---
title: 三层 Monorepo 的 CI/CD 实战：根仓 Tag 驱动 + 云效 Flow + buildah + 双机房 Compose 部署
date: 2026-08-27
tags:
  - cicd
  - 云效
  - docker
  - monorepo
  - 服务器运维
  - deploy
status: completed
---

# 三层 Monorepo 的 CI/CD 实战：根仓 Tag 驱动 + 云效 Flow + buildah + 双机房 Compose 部署

> 以 jiyacr 项目的生产级 CI/CD 链路为样本，完整拆解一套「Git tag 触发 → 云效 Flow 构建推镜像 → 人工审批 → 服务器 Compose 部署 + 健康检查 + 回滚」的设计与实现。重点不在 YAML 怎么抄，而在**每个设计决策背后的坑**——这套链路里几乎每一条规则都是踩坑踩出来的。

## 一、要解决什么问题

项目背景决定了 CI/CD 的形态，先看三个硬约束：

**1. 三层嵌套子模块的仓库拓扑。** 不是普通 Monorepo，而是「根仓 → 框架仓（fork 的上游框架）→ 业务仓」三层 git submodule 嵌套：

```text
jiyacr (根仓，发布锚点)
├── apps/admin-api   框架仓(ruoyi-vue-plus fork)
│   └── ruoyi-modules/ild-flash-backend    业务仓(后端)
├── apps/admin-web   框架仓(vben fork)
│   └── apps/ild-flash-admin              业务仓(管理端)
├── apps/web         框架仓(jiya-web workspace)
│   ├── apps/ild-flash-web                业务仓(前台)
│   └── apps/portal-web                   业务仓(门户)
├── apps/opensource/gpt-image-studio       开源副本仓(根仓直属)
├── deploy           部署仓(compose/流水线YAML/gateway/infra)
└── docs             文档仓
```

**2. 一个仓里跑多个可独立发布的项目。** 主站 ild-flash（后端+管理端+前台三镜像）、门户 portal、网关 gateway、开源站 image、商城 mall，各自的发布节奏不同，不能「推 main 就全量发布」。

**3. 双机房。** ild-flash / gateway 部署在国内 ECS，portal / image / mall 部署在美国 ECS，两边的编排文件、TLS 终结方式、主机组都不同。

由此推导出整套设计的两个支点：

- **发布单元 = 根仓的项目前缀 tag**（`ild-flash-v1.5.2`），一个 tag 锚定三层子模块指针，全项目同版本；
- **CI 全自动、CD 人工卡点**——打 tag 即构建推送镜像，部署前由人工审批放行。

## 二、全景架构

```text
开发者: 业务仓 dev 分支写代码 → 提交自底向上(业务仓→框架仓→根仓)
   │
   ▼ 发布前
scripts/sync-pointers.sh  业务 dev→main 合并推送 → 逐级推指针(ls-remote 校验)
   │
   ▼
根仓 main 打 tag:  ild-flash-v1.5.2  →  push
   │
   ▼ Codeup webhook (flow-openapi.aliyun.com)
云效 Flow 流水线 (tagPush 触发, branchesFilter 正则匹配前缀)
   │
   ├─ prep_stage     解析版本 + 反查 deploy 子模块指针 → deploy-meta.txt 制品
   ├─ build_stage    buildah 并行构建 linux/amd64 镜像 → 推 ACR 个人版 jiyacr/*:<版本>
   ├─ approve_stage  ManualValidate 人工卡点(审批人放行)
   └─ deploy_stage   VMDeploy → 服务器 git checkout DEPLOY_REF → deploy-release.sh
                          │
                          ▼
                     更新 .env 镜像 tag → compose pull/up → 逐 URL 健康检查
                          │
                          ├─ 失败 → dump 日志 / 可选自动回滚
                          └─ 成功 → 写 .last-deployed-<project> 状态文件
```

## 三、仓库拓扑与指针纪律：为什么发布前必须 sync-pointers

三层 submodule 带来一个普通单仓没有的风险：**根仓 tag 锚定的 gitlink 必须全部远端可达**，否则流水线的递归检出（`submodule: true`）会在某一层拉到不存在的 commit，构建直接失败，甚至克隆出一个「业务仓是旧代码」的镜像。

所以业务提交必须是自底向上的：

1. 业务仓提交并推送；
2. 框架仓提交更新后的 gitlink 并推送；
3. 根仓提交框架仓 gitlink。

发布前用脚本固化这个流程（`sh scripts/sync-pointers.sh`）：业务仓 `dev → main` 合并推送 → 框架仓指针提交推送（每步 `ls-remote` 校验远端可达）→ 根仓指针提交（不推根仓，推送是人工决策）。

> **教学经验**：子模块越深，「指针纪律」越要脚本化。靠人肉记顺序必然出错，而出错的代价是发布了错误的代码。

## 四、发布入口：项目前缀 Tag

### 4.1 为什么是 tag 而不是分支

多项目共仓时「推 main 触发发布」不可行——改一行 portal 代码不该发布主站。tag 天然是「一次发布」的语义快照，加上前缀做路由：

| tag 前缀 | 触发流水线 | 构建产物（ACR `jiyacr/*`） | 部署目标 |
| --- | --- | --- | --- |
| `ild-flash-v*` | 5204303 | ild-flash-backend / ild-flash-admin / ild-flash-web（三镜像并行） | 国内机 |
| `portal-v*` | 5204304 | portal-web | 美国机 |
| `gateway-v*` | 5204305 | gateway（nginx 配置打进镜像） | 国内机 |
| `image-v*` | 5220979 | image-web + companion（同一 Dockerfile 双 target） | 美国机 |
| `mall-v*` | 5221666 | mall-admin + lamp-web | 美国机 |

### 4.2 触发机制与版本号格式

流水线 source 配置：

```yaml
sources:
  jiyacr_src:
    type: codeup
    name: jiyacr
    endpoint: https://codeup.aliyun.com/.../jiyacr.git
    branch: main
    submodule: true                        # 三层递归检出
    triggerEvents:
      - tagPush                            # 仅 tag 触发，推 main 不触发
    branchesFilter: ^(ild-flash-v\d+\.\d+\.\d+|main)$
```

三个要点：

1. **触发的本体是 Codeup 仓库上的 webhook**（指向 `flow-openapi.aliyun.com`，创建流水线时自动生成，每条 tag 触发的流水线对应一个）。**千万不要在 Codeup 仓库设置的 Webhooks 页删掉它**，删了发布链路即断，而且很难排查——流水线页面看不出异常。
2. **版本号必须是 `X.Y.Z` 纯数字**。`ild-flash-v1.5.2-rc1` 这种带后缀的 tag 不匹配 `branchesFilter` 正则，会**静默不触发**——没有任何报错，你以为发了其实没发。
3. 正则里放行 `main` 是为了流水线手动调试运行，不是发布路径。

## 五、流水线四阶段逐段剖析

以 ild-flash 流水线（`deploy/pipelines/ildflash-build-publish.yaml`）为样本。

### 5.1 prep_stage：版本解析与部署锚点

```yaml
prep_job:
  runsOn:
    group: public/cn-beijing
    container: build-steps-public-registry.../alinux3:latest
  steps:
    meta_step:
      step: Command
      with:
        run: |
          set -e
          export VERSION="${CI_COMMIT_REF_NAME#ild-flash-v}"
          echo "$VERSION" | grep -Eq '^[0-9]+\.[0-9]+\.[0-9]+$' || exit 1
          export DEPLOY_REF=$(git ls-tree HEAD deploy | awk '{print $3}')
          [ -n "$DEPLOY_REF" ] || { echo "DEPLOY_REF 为空" >&2; exit 1; }
          cat > deploy-meta.txt <<EOF
          PROJECT=ild-flash
          VERSION=$VERSION
          DEPLOY_REF=$DEPLOY_REF
          EOF
    upload_step:
      step: ArtifactUpload                    # 上传为流水线制品
      with:
        repo: flow_generic_repo
        artifact: deploy-meta
        version: meta-ildflash-${DATETIME}
        filePath: deploy-meta.txt
```

这一段做了两件教学设计上都值得抄的事：

**① DEPLOY_REF：让部署编排跟代码同版。** `git ls-tree HEAD deploy` 取的是**这个 tag 锚定的 deploy 子模块 commit**。部署阶段服务器会 checkout 到这个 commit 再执行部署脚本——保证了「发布用的 compose 编排/部署脚本」与「发布的代码」严格来自同一个快照，不会出现「部署脚本是新版、镜像是旧版」的错位。

**② 版本元数据走制品（Artifact），不走环境变量。** 云效的 FLOW_ENV 环境变量**不能跨主机组传递**（实测结论），而构建 job 和部署 job 跑在不同主机组（公共构建集群 vs 你的 ECS 主机组）。制品是唯一可靠的跨 stage、跨主机组通道。这是云效 Flow 的一个隐性约束，自己摸索很容易在这里卡住。

### 5.2 build_stage：为什么用 buildah 而不是 docker

三个镜像（backend / admin / web）各一个 job **并行构建**，每个 job 的核心命令：

```bash
echo "$ACR_PASSWORD" | docker login --username "$ACR_USERNAME" --password-stdin "$ACR_REGISTRY"
which buildah >/dev/null 2>&1 || yum install -y -q buildah
export BUILDAH_ISOLATION=chroot
buildah --storage-driver=vfs bud --format docker --platform linux/amd64 \
  -f apps/admin-api/ruoyi-modules/ild-flash-backend/Dockerfile \
  -t "$ACR_REGISTRY/jiyacr/ild-flash-backend:$VERSION" \
  apps/admin-api                                  # 注意:构建上下文是框架仓根
buildah --storage-driver=vfs push \
  "$ACR_REGISTRY/jiyacr/ild-flash-backend:$VERSION" \
  "docker://$ACR_REGISTRY/jiyacr/ild-flash-backend:$VERSION"
```

这段脚本里每个看似多余的参数都对应一个坑：

| 细节 | 原因 |
| --- | --- |
| 用 `buildah` 而非 `docker build` | 云效公共构建集群**没有 docker daemon** |
| 不用官方 `ACRDockerBuild` 步骤 | 该步骤在此环境下**无声失败**（历史问题），表现为流水线绿但镜像没更新 |
| `--storage-driver=vfs` | 无特权容器内 overlayfs 不可用，vfs 是最兼容的兜底（慢但稳） |
| `BUILDAH_ISOLATION=chroot` | 无 rootless 命名空间支持时 chroot 隔离可用性最好 |
| `--platform linux/amd64` | 本地开发机是 Apple Silicon（arm64），**基础镜像必须推 amd64**——曾因从 Mac 推了 arm64 基础镜像导致全量构建失败 |
| 构建上下文 = `apps/admin-api`（框架仓根） | 三层拓扑的构建语义：业务仓无独立构建，Dockerfile 在业务仓内但上下文必须是框架仓（要能 COPY 框架层源码） |

凭证管理：ACR 账密放在云效**加密变量组**（`jiyacr-acr`），YAML 里只有 `$ACR_USERNAME` 引用，无明文。

另一个细节：**镜像 tag 只用纯版本号**（`jiyacr/ild-flash-backend:1.5.2`），不搞 `latest`。部署侧 `.env` 里每个服务有独立的 `*_IMAGE_TAG` 变量，版本推进是显式的。

### 5.3 approve_stage：人工卡点的正确写法

```yaml
approve_stage:
  name: 部署审批(人工卡点)
  jobs:
    validate_job:
      component: ManualValidate
      with:
        validatorType: users
        validateMethod: and
        validators:
          - 1295277249165417        # 审批人阿里云 ID
```

**这是全套链路里教训最贵的一处。** 之前用的是 stage 上的 `driven: manual` 字段——这个字段**不是云效官方 YAML 语法，引擎会静默忽略**。结果是：以为有审批拦截，实际打 tag 即自动部署，v1.5.3 之前所有版本都在「裸奔」。直到某次核对运行记录才发现。

修复后改用官方 `ManualValidate` 组件：有服务端校验、运行记录可查、审批人可在 Flow 控制台或钉钉卡片上放行。

> **教学经验**：对「静默忽略」类配置保持警惕。任何平台的关键控制项（审批、拦截、权限），配置完后**必须实测一次反向场景**（不审批会不会真的不部署），而不是配置完就当生效了。只构建不部署时，放着不批即可，构建产物保留在 ACR 随时可发。

### 5.4 deploy_stage：VMDeploy 到自有 ECS

```yaml
deploy_job:
  component: VMDeploy
  with:
    artifact: $[stages.prep_stage.prep_job.upload_step.artifacts.deploy-meta]  # 引用制品
    machineGroup: v8nkI9jsVxF3eZJJ          # 国内主机组(美国机是另一个 ID)
    executeUser: yihongda
    artifactDownloadPath: /home/yihongda/deploy-meta.tgz
    run: |
      set -e
      mkdir -p /tmp/jiyacr-meta && tar zxf /home/yihongda/deploy-meta.tgz -C /tmp/jiyacr-meta
      . /tmp/jiyacr-meta/deploy-meta.txt                       # source 出 PROJECT/VERSION/DEPLOY_REF
      cd /home/yihongda/jiyacr-deploy
      git fetch origin && git checkout -q "$DEPLOY_REF"        # 部署仓对版
      cd compose
      PROJECT="$PROJECT" VERSION="$VERSION" sh ./deploy-release.sh
```

设计要点：

- **部署仓本身就是 git 仓库克隆在服务器上**（`/home/yihongda/jiyacr-deploy`），取代了旧的 rsync 同步。改 compose/infra 文件不需要发版，服务器 `git pull` 即可；发版时则 checkout 到 DEPLOY_REF 对版。服务器用的是**只读部署密钥**（deploy key），不是个人 PAT——凭证最小权限。
- **版本信息不靠猜**。早期版本有「从 ACR 查最新 tag 猜版本」的逻辑，已退役——版本一律来自显式入参（tag 或手动参数），杜绝「部署了意外版本」。

## 六、部署脚本 deploy-release.sh：安全网全在这里

流水线只负责把脚本送上服务器并执行，真正的发布安全性由 `deploy/compose/deploy-release.sh` 保证。逐层拆解：

### 6.1 项目 → 服务/健康检查的映射表

```sh
case "$PROJECT" in
  ild-flash)
    SERVICES="backend user-web admin-web"
    ENV_KEYS="BACKEND_IMAGE_TAG USER_WEB_IMAGE_TAG ADMIN_WEB_IMAGE_TAG"
    HEALTH_URLS="ildflash.com|https://ildflash.com/ ... api-business|https://api.ildflash.com/v1/public/home-summary?..."
    ;;
  gateway)
    SERVICES="gateway"
    HEALTH_URLS="gateway-443-local|https://ildflash.com/ ..."
    ;;
esac
```

一个脚本服务全部项目，差异收敛在映射表里。注意健康检查不只看首页——ild-flash 还检查了一个真实业务 API（`/v1/public/home-summary`），因为「容器活着」不等于「业务可用」。

### 6.2 部署前的三道防线

```sh
# 防线一:磁盘水位
FREE_MB=$(df -Pm / | awk 'NR==2 {print $4}')
[ "$FREE_MB" -ge "$MIN_FREE_MB" ] || fail "磁盘余量不足,拒绝部署"   # 默认 5GB

# 防线二:ild-flash 部署前 mysqldump(保留最近 5 份)
sudo docker exec ild-flash-mysql sh -c 'exec mysqldump ... --databases ry-vue' | gzip > "$BACKUP_FILE"

# 防线三:记录旧版本(状态文件优先,运行容器镜像名兜底)
OLD_VERSION=$(cat "$STATE_FILE" 2>/dev/null)
[ -z "$OLD_VERSION" ] && OLD_VERSION=$(sudo docker inspect ... | grep -oE '[0-9]+\.[0-9]+\.[0-9]+$')
```

磁盘水位防线源于真实教训：镜像堆积把磁盘打满后，MySQL 数据目录损坏过一次（虽零数据丢失恢复，但从此部署前必查水位 + 必备份）。

### 6.3 部署与健康检查

```sh
set_env_versions "$VERSION"          # sed 更新 .env 里的 *_IMAGE_TAG
$dc config >/tmp/jiyacr-compose-config.yml || fail "compose config 校验失败"
$dc pull $SERVICES || fail "镜像拉取失败"
$dc up -d $SERVICES  || fail "容器启动失败"

# 健康检查:curl --resolve 打本机 443,绕过 DNS/CDN 直击本机
curl -fsS --resolve "${host}:443:127.0.0.1" "$url"
```

两个技巧值得单独说：

- **`curl --resolve host:443:127.0.0.1`**：健康检查从服务器本机发起，用 `--resolve` 把域名钉到本机回环——这样检查的是「本机这套容器是否真实服务了 HTTPS」，而不是「公网链路是否通畅」。公网链路有 CDN/运营商等不可控因素，不该决定发布成败。
- **先打 gateway 本机 443 快速检查**：网关挂了则所有业务域名全不可达，先快速失败比逐个等超时省几分钟。

### 6.4 失败处置：dump 日志 + 可选自动回滚

```sh
if [ "$HEALTH_OK" != "true" ]; then
  dump_failure_logs                              # 各容器日志尾部 + compose ps
  if [ "$ALLOW_ROLLBACK" = "true" ] && [ -n "$OLD_VERSION" ]; then
    set_env_versions "$OLD_VERSION"              # 回滚 = 把 .env 改回旧版本重来一遍
    $dc pull $SERVICES && $dc up -d $SERVICES
  else
    echo "保持失败现场待人工处置"                  # 默认不自动回滚
    echo "(若含数据库迁移,注意可能是旧代码+新 schema 混合态,先查 ild_flyway_schema_history)"
  fi
  exit 1
fi
echo "$VERSION" > "$STATE_FILE"                  # 成功才写状态文件
```

设计上默认**不自动回滚**（`ALLOW_ROLLBACK=false`），理由是：回滚只回滚镜像，**不回滚数据库 schema**（Flyway 迁移已执行）。盲目自动回滚可能制造「旧代码 + 新 schema」的混合态，比挂着更危险。失败时保留现场、dump 日志，由人判断。

> **教学经验**：自动回滚的前提是「回滚操作本身安全」。涉及有状态变更（DB 迁移）时，保守的默认行为 + 充分失败的现场信息，比激进的自动回滚更可靠。

## 七、回滚与指定版本部署：手动流水线

紧急回滚走独立的手动流水线（国内 5204306 / 美国机 5220980），不依赖 tag 触发：

```yaml
variables:
  - key: PROJECT
    value: ild-flash
  - key: VERSION
    value: 1.5.0
# ...
meta_step:
  run: |
    git fetch origin --tags
    TAG="${PROJECT}-v${VERSION}"
    DEPLOY_REF=$(git ls-tree "$TAG" deploy | awk '{print $3}')   # 由 tag 反查部署锚点
    [ -n "$DEPLOY_REF" ] || { echo "反查失败:tag 不存在"; exit 1; }
```

注意这里没有构建阶段——镜像早已在 ACR 里（发布即制品），回滚只是把指定版本的镜像重新跑起来。反查 DEPLOY_REF 保证回滚用的编排也与那个版本一致。取不到 tag 直接拒绝执行，防止手滑输错版本号。

这也回答了「镜像仓库里 tag 越来越多怎么办」——那正是设计意图：**每一个历史版本都是可回滚的锚点**。

## 八、双机房：同一套链路，两个落点

| | 国内机 | 美国机 |
| --- | --- | --- |
| 项目 | ild-flash / gateway | portal / image / mall |
| 主机组 ID | `v8nkI9jsVxF3eZJJ` | `nWReGlSJMUBN6vM8` |
| 编排文件 | `docker-compose.prod.yml` | `docker-compose.us.yml` |
| TLS | gateway 容器（nginx 镜像）终结 | **宿主机 nginx + certbot webroot** 终结，无 gateway 容器 |
| 基础设施 | MySQL / Redis 容器 | 无（跨洋调国内 API） |

差异通过两个机制收敛进同一套链路：流水线的 `machineGroup` 决定部署到哪台机器；部署命令传 `COMPOSE_FILE=docker-compose.us.yml` 决定用哪份编排。deploy-release.sh 的映射表对两边通用。

mall 项目体现了一个取舍：后端不单独发版，**随国内 ild-flash-v\* 的 ruoyi-admin 镜像一起发布**，美国机只跑前台两个容器——前台跨洋调 `api.ildflash.com`。跨机房架构里「哪些服务必须同机房、哪些可以跨洋」是设计时就要定死的，CI/CD 只是如实反映这个决策。

## 九、进阶设计点

### 9.1 同一 Dockerfile 双 target 出双镜像

image 项目（image.ildflash.com 前端 + companion 后端）共用一个 `Dockerfile.ild`，流水线用 `--target` 出两个镜像：

```bash
buildah bud --target web       -f apps/opensource/gpt-image-studio/Dockerfile.ild -t .../image-web:$VERSION ...
buildah bud --target companion -f apps/opensource/gpt-image-studio/Dockerfile.ild -t .../companion:$VERSION ...
```

前后端同一个 tag 版本化，一次发布双镜像，版本永远对齐。

### 9.2 流水线 YAML 即代码，本地是唯一真相

所有流水线 YAML 落盘在 `deploy/pipelines/`，云效控制台只是镜像。改了 YAML 后用脚本同步：

```sh
aliyun devops flow-update-pipeline --pipeline-id "$id" --name "$name" --content "$(cat $file)"
aliyun devops flow-add-pipeline-relations --pipeline-id "$id" \
  --rel-object-type VARIABLE_GROUP --rel-object-ids "43281"   # 加密变量组单独挂接(幂等)
```

好处：YAML 走 git 评审、有历史、可 diff；坏处：改了本地忘同步会造成「仓里和控制台不一致」，需要纪律。**注意加密变量组不在 YAML 里**，是同步时单独挂接的。

### 9.3 基础设施不跟业务发布

MySQL / Redis 容器不在任何业务流水线的部署范围内，且镜像版本**钉死**（mysql `8.0.46`、redis `7.4.9-alpine`）——浮动 tag（如 `mysql:8.0`）会在容器重建时隐性升级，数据库版本漂移是生产事故的经典来源。数据库结构变更由 Flyway 随后端发布接管（`V*.sql` 只增不改）。

### 9.4 发版操作的 AI 半自动化

发版流程固化成了 ZCode skill（`.zcode/skills/release`）：跑 sync-pointers → 读最新 tag 算出 patch/minor/major 三选项 → 用户在确认弹窗点选版本号（点选 = 同意打 tag + push）→ 打 tag 并 `ls-remote` 实测确认 → 汇报触发的流水线 ID。把「铁律」（先同步指针、失败就停、不 force push tag）写成机器可执行的检查，比写在文档里靠人肉遵守可靠得多。

## 十、踩坑清单（全文最有价值的一节）

| 坑 | 现象 | 对策 |
| --- | --- | --- |
| Mac 推 arm64 基础镜像 | amd64 服务器/CI 构建全挂 | 基础镜像必须 `--platform linux/amd64` 构建推送 |
| 官方 ACRDockerBuild 步骤无声失败 | 流水线绿但镜像没更新 | 弃用，改 Command + buildah |
| `driven: manual` 非官方语法被静默忽略 | 以为有审批，实际打 tag 即自动部署 | 用官方 `ManualValidate` 组件；关键控制项必须实测反向场景 |
| FLOW_ENV 不跨主机组 | 部署 job 拿不到构建 job 的变量 | 版本元数据走制品（ArtifactUpload/下载） |
| tag 带 rc 后缀不匹配触发正则 | **静默不触发**，无任何报错 | 版本号钉死 `X.Y.Z`；触发正则与版本规范写进检查清单 |
| 删除 Codeup webhook | 发布链路断且难排查 | 文档标注勿删；webhook 数量纳入巡检 |
| 磁盘打满 | MySQL 数据目录损坏（已恢复） | 部署前磁盘水位检查 + mysqldump，保留 5 份 |
| 浮动基础镜像 tag | 容器重建时隐性升级 | mysql/redis 版本钉死 |
| 服务器裸 `compose up -d` | 拉取未上线服务的占位 tag（`0.0.0-not-deployed`）导致整条命令失败 | 手工操作必须带服务清单 |
| bash 语法在 dash 下被拒 | 部署脚本在 sh 环境执行失败 | 脚本一律 POSIX 写法（禁 `[[ ]]`、`pipefail` 等） |
| 业务仓改依赖未更新锁文件 | CI `--frozen-lockfile` 必挂 | 发布前在宿主仓重新生成 pnpm 锁文件并提交 |
| 用个人 PAT 做服务器拉取认证 | 权限过大、泄露面大 | 换绑仓库级只读 deploy key |

## 十一、把这套设计搬到你的项目：复用清单

如果你要在自己的项目复制这套链路，按此顺序落地：

1. **定发布单元**：一个仓几个项目？每个项目一个 tag 前缀 + 一条流水线，正则钉死版本格式。
2. **理清仓库拓扑**：有 submodule 就先写指针同步脚本，并在流水线开递归检出。
3. **搭构建**：公共集群无 docker daemon 就用 buildah；基础镜像钉 amd64；构建上下文选对层级。
4. **建制品通道**：跨 stage/主机组的元数据走 Artifact，别信环境变量。
5. **加人工卡点**：用官方审批组件，**配置完实测一次不审批的场景**。
6. **写部署脚本**：项目映射表 + 部署前防线（磁盘/备份/记录旧版）+ 本机健康检查（`curl --resolve`）+ 失败现场保护（默认不自动回滚）+ 状态文件。
7. **建回滚通道**：手动流水线 + 按 tag 反查部署锚点 + 取不到拒绝执行。
8. **凭证最小化**：加密变量组存 registry 账密；服务器用只读 deploy key；YAML 零明文。
9. **YAML 落盘进 git**：本地为唯一真相 + CLI 同步脚本，改动可追溯。
10. **写踩坑文档**：每个坑记录现象与对策，这是团队最值钱的资产。

## 十二、参考

项目内的权威文档（jiyacr 仓）：

- `docs/operations/cicd-overview.md` — CI/CD 总览（本文的源头）
- `docs/operations/release-checklist.md` — 发布检查清单
- `docs/operations/server-layout.md` — 双机房服务器部署结构
- `deploy/pipelines/*.yaml` — 全部流水线定义
- `deploy/compose/deploy-release.sh` — 部署脚本
- `.zcode/skills/release/SKILL.md` — AI 半自动发版 skill
- `docs/operations/incident-20260816-first-release-mysql-recovery.md` — 首次发布 MySQL 故障事件记录

---

> 一句话总结这套设计：**用 tag 做发布锚点，用制品做元数据通道，用人工卡点守住部署闸门，用部署脚本兜底一切意外，用踩坑文档让教训不重复发生。**
