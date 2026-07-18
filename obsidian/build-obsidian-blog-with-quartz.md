---
title: 用 Quartz + GitHub Actions 搭建 Obsidian 笔记自动发布博客
date: 2026-07-18
tags:
  - obsidian
  - quartz
  - blog
  - github-actions
  - ci-cd
  - tutorial
status: in-progress
---

# 用 Quartz + Quartz + GitHub Actions 搭建 Obsidian 笔记自动发布博客

> [!info] 本指南能帮你做什么
> 跟着做完，你会得到一个**自动更新的博客**：在 Obsidian 写笔记 push 后，博客 1-2 分钟内自动更新。架构如下：
>
> - **笔记仓**：你的 Obsidian vault，只管写笔记
> - **框架仓**：Quartz 配置 + CI 部署，笔记以 git submodule 引入
> - **双分支**：`main`（业务层）+ `v5`（上游镜像层，方便升级）
>
> 最终效果参考：[blog.honlnk.com](https://blog.honlnk.com)

## 前置准备

开始前你需要准备：

| 准备项 | 说明 |
|---|---|
| GitHub 账号 | 两个仓库都托管在 GitHub |
| Obsidian vault | 已经在用 Obsidian 记笔记（或有笔记准备发布） |
| 一个域名（可选） | 不用自定义域名也能用 `*.github.io` 默认地址 |
| 本地装好 git + Node.js 22+ | 构建和推送用 |

> [!tip] 命令里的占位符
> 本指南命令中用占位符代表"你要换成自己的"：
> - `<你的用户名>` → 比如 `honlnk`
> - `<笔记仓>` → 比如 `my-notes`
> - `<框架仓>` → 比如 `my-blog-site`
> - `<你的域名>` → 比如 `blog.example.com`（没有就跳过域名相关步骤）

## 第 1 步：准备笔记仓

如果你已经有 Obsidian vault 并推到了 GitHub，跳到第 2 步。

```bash
# 1. 在 GitHub 新建一个空仓库（比如 my-notes），不要勾 README/gitignore
# 2. 把本地 vault 初始化为 git 仓库并推送
cd ~/你的/vault/路径
git init
git add .
git commit -m "init: 初始化笔记仓"
git branch -M master
git remote add origin git@github.com:<你的用户名>/<笔记仓>.git
git push -u origin master
```

> [!warning] 笔记仓根目录不要放 index.md
> 笔记仓的首页（`index.md`）由框架仓提供（第 5 步）。如果笔记仓根目录也有 `index.md`，会冲突。如果你 vault 根目录有同名文件，重命名或移到子目录。

## 第 2 步：初始化框架仓（基于 Quartz v5）

```bash
# clone Quartz 上游
git clone https://github.com/jackyzha0/quartz.git <框架仓>
cd <框架仓>

# 把 origin 改成你自己的仓库，upstream 保留指向 Quartz 官方
git remote rename origin upstream
git remote add origin git@github.com:<你的用户名>/<框架仓>.git

# 在 GitHub 新建空仓库 <框架仓>（不勾 README），然后推送
git push -u origin v5

npm install
```

初始化 Quartz 配置（按提示选）：

```bash
npx quartz create \
  --template default \
  --strategy new \
  --baseUrl <你的域名>           # 没有域名就填 <用户名>.github.io/<框架仓>
  --links shortest
```

## 第 3 步：建立 main / v5 双分支

保持 `v5` 作为"上游镜像层"（只接收 Quartz 官方更新），把所有业务改动放到 `main`：

```bash
# 当前在 v5 分支（来自上游 clone）
# 1. 改名为 main（保留刚才 npx quartz create 的改动）
git branch -m v5 main
git push -u origin main

# 2. 从上游重建一个纯净的 v5 分支（镜像层）
git branch v5 upstream/v5
git push -u origin v5

# 3. 把 GitHub 默认分支改成 main
gh repo edit <你的用户名>/<框架仓> --default-branch main

# 4. 后续业务改动都在 main 上做
git checkout main
```

> [!note] 两个分支的职责
> - **`v5`**：上游镜像层，保持和 Quartz 官方一致，**不直接写业务**
> - **`main`**：业务层，所有定制（配置、CI、首页）都在这里提交

## 第 4 步：把笔记仓加为 submodule

```bash
git checkout main   # 确保在 main 分支

# 把笔记仓作为 submodule 挂到 content/notes/
git submodule add https://github.com/<你的用户名>/<笔记仓>.git content/notes
```

这会生成 `.gitmodules` 文件。主仓只记录 submodule 的 commit 指针，**笔记内容不进入框架仓的 git 历史**。

## 第 5 步：创建首页 `content/index.md`

Quartz 默认只为 `content/` 根目录生成简陋的 folder listing。写一个 `content/index.md` 可以自定义首页：

```markdown
---
title: 我的博客
---

欢迎来到我的博客。

## 内容主题

- **主题 A** — 简要描述
- **主题 B** — 简要描述

## 如何浏览

- 顶部搜索框支持全文搜索
- 左侧文件树按目录浏览
- 右侧图谱展示笔记间的关联
```

## 第 6 步：配置站点 `quartz.config.yaml`

打开 `quartz.config.yaml`，修改关键字段：

```yaml
configuration:
  pageTitle: 我的博客                    # 站点标题
  pageTitleSuffix: ""                   # 标题后缀（可留空）
  baseUrl: <你的域名>                    # 和第 2 步一致
  locale: zh-CN                         # 界面语言
  defaultDateType: modified             # 按 git 最后修改时间排序
  ignorePatterns:                       # 不发布的文件/目录
    - private
    - templates
    - .obsidian
    - .smtcmp*                          # Obsidian smart-composer 缓存
    - README.md                         # 仓库说明文件不发布
    - README.en.md
    - LICENSE*
    - folder-alias.json                 # Obsidian 插件配置
```

> [!note] ignorePatterns 跨层匹配
> 这些规则会自动应用到 `content/notes/` 子目录下。所以笔记仓根目录的 README/LICENSE 也会被排除，不用为子目录单独配置。

## 第 7 步：编写 CI 部署工作流

创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy blog to GitHub Pages

on:
  push:
    branches: [main]                          # 框架仓自身改动触发
  repository_dispatch:
    types: [notes-updated]                    # 笔记仓 push 联动触发
  workflow_dispatch:                          # 手动触发

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # 1. checkout 框架仓 + 初始化 submodule
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
          submodules: recursive

      # 2. 把 submodule 更新到笔记仓 master 最新
      - name: Update notes submodule to latest
        run: |
          git -C content/notes fetch --depth 1 origin master
          git -C content/notes checkout origin/master

      # 3. 构建
      - uses: actions/setup-node@v7
        with:
          node-version: 22
      - run: npm ci
      - uses: actions/cache@v6
        with:
          path: .quartz/plugins
          key: quartz-plugins-${{ hashFiles('quartz.lock.json') }}
      - run: npx quartz plugin install --from-config
      - run: npx quartz build

      # 4. 写入自定义域名（没有自定义域名就删掉这步）
      - run: echo "<你的域名>" > public/CNAME
      - uses: actions/upload-pages-artifact@v5
        with:
          path: public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v5
```

## 第 8 步：配置 GitHub Pages

```bash
# 1. 把 Pages source 设为 GitHub Actions
gh api --method POST repos/<你的用户名>/<框架仓>/pages \
  -f build_type=workflow

# 2. 提交所有改动并推送
git add -A
git commit -m "feat: 初始化博客框架（Quartz + submodule + CI）"
git push origin main
```

推送后会自动触发首次部署。等 1-2 分钟，访问：

```bash
# 查看部署状态
gh run watch --repo <你的用户名>/<框架仓>

# 部署成功后访问（换成你的地址）
# 有自定义域名：https://<你的域名>
# 无自定义域名：https://<你的用户名>.github.io/<框架仓>
```

## 第 9 步：配置笔记仓联动 trigger（实现自动更新）

到目前为止，改框架仓才会触发部署。这一步让**笔记仓 push 后自动触发博客重建**。

### 9.1 创建 GitHub Token

打开 https://github.com/settings/personal-access-tokens/new ，创建 Fine-grained token：

| 字段 | 值 |
|---|---|
| Token name | `blog-site-dispatch`（随意） |
| Expiration | `1 year` |
| Repository access | Only select repositories → 勾选**框架仓**（不是笔记仓） |
| Permissions → Contents | **Read and write** ⭐ |

> [!warning] 权限必须是 Contents，不是 Actions
> `repository_dispatch` 端点归属 Contents 权限域。如果给 Actions 权限（哪怕 Read and write），dispatch 会返回 403。权限条右侧有下拉，默认 Read-only，要改成 Read and write。

生成后复制 token（只显示一次）。

### 9.2 把 Token 存为笔记仓的 Secret

```bash
# 用 gh CLI 直接设置（会提示粘贴 token）
gh secret set BLOG_SITE_PAT --repo <你的用户名>/<笔记仓>
```

或网页操作：`笔记仓 Settings → Secrets and variables → Actions → New repository secret`，Name 填 `BLOG_SITE_PAT`，Value 粘贴 token。

### 9.3 在笔记仓添加 trigger workflow

在笔记仓创建 `.github/workflows/trigger-site.yml`：

```yaml
name: 触发博客重建

on:
  push:
    branches: [master]

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - name: 通知框架仓重建
        run: |
          RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
            -H "Authorization: Bearer ${{ secrets.BLOG_SITE_PAT }}" \
            -H "Accept: application/vnd.github+json" \
            https://api.github.com/repos/<你的用户名>/<框架仓>/dispatches \
            -d '{"event_type":"notes-updated"}')
          if [ "$RESPONSE" != "204" ]; then
            echo "::error::触发失败 HTTP $RESPONSE（检查 BLOG_SITE_PAT）"
            exit 1
          fi
          echo "✓ 已通知框架仓重建博客"
```

提交并推送到笔记仓：

```bash
cd ~/你的/vault/路径
git add .github/workflows/trigger-site.yml
git commit -m "ci: 新增博客联动 trigger"
git push origin master
```

## 第 10 步：验证整条联动

在笔记仓随便改一篇笔记（加个空格也行），commit & push：

```bash
git add .
git commit -m "test: 验证联动"
git push origin master
```

等 1-2 分钟，检查两个仓库的 Actions 是否都变绿：

```bash
# 笔记仓的 trigger workflow
gh run list --repo <你的用户名>/<笔记仓> --limit 1

# 框架仓的 deploy（触发方式应该显示 repository_dispatch）
gh run list --repo <你的用户名>/<框架仓> --limit 1
```

两个都成功后，访问博客确认内容更新。**至此整套自动化就跑通了**。

## 自定义域名（可选）

如果你有自己的域名：

```bash
# 1. 在域名 DNS 添加 CNAME 记录：
#    <子域名> → <你的用户名>.github.io.
#    （Cloudflare 用户注意：该记录必须设为 DNS only / 灰云，否则 HTTPS 证书签发失败）

# 2. deploy.yml 已经在 build 末尾写了 CNAME 文件（第 7 步的第 4 小步）
#    确认 echo 的内容是你的域名

# 3. 等一次部署完成，GitHub 会自动签发 HTTPS 证书
```

## 日常使用

整套搭好后，日常只需要两件事：

| 场景 | 操作 |
|---|---|
| **写博客** | 在 Obsidian 写笔记 → commit & push 笔记仓 → 1-2 分钟后博客自动更新 |
| **改框架配置**（改主题/标题/CI） | 在框架仓 `main` 分支改 → push → 自动重新部署 |

不用手动触发任何东西。

## 本地预览

改 Quartz 配置或调试时，本地预览：

```bash
cd <框架仓>
npm install
git submodule update --init --recursive        # 初始化笔记 submodule

# 想拉笔记仓 master 最新（而非锁定的 commit）
git -C content/notes fetch origin master
git -C content/notes checkout origin/master

npx quartz build --serve   # → http://localhost:8080
```

## 升级 Quartz

走 v5 → main 两层缓冲：

```bash
cd <框架仓>
git fetch upstream

# 1. 在 v5（上游镜像层）合并上游更新
git checkout v5
git merge upstream/v5
git push origin v5

# 2. 切到 main，把更新过的 v5 合进来
git checkout main
git merge v5
npm install                    # 依赖可能变了
npx quartz build --serve       # 本地验证

# 3. 验证通过后推送，触发部署
git push origin main
```

## 故障排查

### 部署后博客没更新

按数据流顺序排查：

```bash
# 1. 笔记仓的 trigger 跑了吗？
gh run list --repo <你的用户名>/<笔记仓> --limit 1
# 报红 → 多半是 BLOG_SITE_PAT 过期或权限错（应为 Contents:write）

# 2. 框架仓的 deploy 触发了吗？
gh run list --repo <你的用户名>/<框架仓> --limit 1
# 没触发 → 手动跑一次：
gh workflow run deploy.yml --repo <你的用户名>/<框架仓>

# 3. deploy 报错了？
gh run view --log-failed --repo <你的用户名>/<框架仓>
```

### 首页显示 XML 而非 HTML

检查 `content/index.md` 是否存在且已入库：

```bash
git ls-files content/index.md
# 空输出 = 没入库，git add content/index.md && git commit && git push
```

### 笔记内容没出现在博客上

```bash
# 1. 笔记仓 push 成功了吗？
git -C content/notes log --oneline -1   # 应该是最新提交

# 2. CI 里 submodule 更新了吗？看 deploy.yml 的日志：
gh run view --repo <你的用户名>/<框架仓> --log | grep -A3 "Update notes submodule"
```

### `deploy` 报 "Branch X is not allowed to deploy to github-pages"

改过默认分支名后，GitHub Pages 的 environment 分支白名单没同步。修复：

```bash
# 查看当前允许的分支
gh api repos/<你的用户名>/<框架仓>/environments/github-pages/deployment-branch-policies \
  --jq '.branch_policies[] | .name'

# 加上新分支
gh api --method POST repos/<你的用户名>/<框架仓>/environments/github-pages/deployment-branch-policies \
  -f name=main
```

## 目录结构速查

```
<框架仓>/
├── quartz.config.yaml        # 站点配置
├── quartz.lock.json          # 插件版本锁
├── content/
│   ├── index.md              # 首页模板（入库）
│   ├── .gitkeep
│   └── notes/                # 笔记仓 submodule
├── .gitmodules               # submodule 配置
├── .github/workflows/
│   └── deploy.yml            # 构建 + 部署
└── quartz/                   # Quartz v5 核心（上游，勿改）
```

| 分支 | 用途 |
|---|---|
| `main`（默认） | 业务层，日常提交，CI 从这构建 |
| `v5` | 上游镜像层，只接收 Quartz 更新 |

## 相关笔记

- [[blog-publish-with-quartz-submodule]] — 完整实现方案（含踩坑记录和设计决策）
- [[git-submodules-nested-repositories-guide]] — git submodule 深入指南
- [[obsidian-quickstart]] — Obsidian 快速上手

## 参考链接

- [Quartz v5 官方文档](https://quartz.jzhao.xyz/)
- [GitHub Actions: repository_dispatch](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows#repository_dispatch)
- [Fine-grained PAT 权限要求](https://docs.github.com/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens)
- [GitHub Pages 部署](https://docs.github.com/pages)
