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

# 用 Quartz + GitHub Actions 搭建 Obsidian 笔记自动发布博客

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
| **GitHub CLI (`gh`)** | 本指南多处用 `gh` 配置 Pages/secrets/environment。先 `brew install gh`（macOS）或见 [官方安装文档](https://cli.github.com/)，再 `gh auth login` 登录 |
| **SSH key 已配置到 GitHub** | 推送命令用 `git@github.com:` 形式。没配 SSH 就把命令里的 `git@github.com:` 全换成 `https://github.com/`（首次 push 会提示输 token） |

> [!tip] 命令里的占位符
> 本指南命令中用占位符代表"你要换成自己的"：
> - `<你的用户名>` → 比如 `honlnk`
> - `<笔记仓>` → 比如 `my-notes`
> - `<框架仓>` → 比如 `my-blog-site`
> - `<你的域名>` → 比如 `blog.example.com`。**没有自定义域名时**，`--baseUrl` 不能省，填 `<你的用户名>.github.io/<框架仓>`（如 `honlnk.github.io/my-blog-site`），并跳过"自定义域名"章节

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

# 自检：确认当前分支（上游默认是 v5，但未来可能变；如果不是 v5，下文所有 v5 换成你看到的分支名）
git branch --show-current

# 把 origin 改成你自己的仓库，upstream 保留指向 Quartz 官方
git remote rename origin upstream
git remote add origin git@github.com:<你的用户名>/<框架仓>.git

# 在 GitHub 新建空仓库 <框架仓>（不勾 README），然后推送
git push -u origin v5

npm install
```

初始化 Quartz 配置（`--strategy new` 保留 Quartz 默认配置不覆盖；`--links shortest` 让笔记间 `[[链接]]` 用最短路径解析）：

```bash
npx quartz create \
  --template default \
  --strategy new \
  --baseUrl <你的域名>           # 没有域名就填 <用户名>.github.io/<框架仓>
  --links shortest
```

> [!note] 这一步生成了什么
> `npx quartz create` 会生成完整的 `quartz.config.yaml`（含 ~40 个社区插件清单、layout 编排、默认主题）和 `quartz.lock.json`（锁版本）。**这些是后续步骤的基础，不要手删 `plugins:` / `layout:` 段**。后续在第 6 步只改 `configuration:` 段的几个字段。

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

> [!warning] 笔记仓是私有的怎么办？
> 如果笔记仓是**私有**的，CI 里 `actions/checkout` 默认的 `GITHUB_TOKEN` 只能读框架仓，读不了另一个私有仓的 submodule，会在第 7 步 checkout 时 403。两个选择：
> - **方案 A（推荐）**：把笔记仓设为公开。Obsidian vault 一般没什么隐私（私人内容用 `.obsidian/` 或 `private/` 目录隔离，已被 `ignorePatterns` 排除）。
> - **方案 B**：笔记仓保持私有，但要在第 7 步 deploy.yml 的 checkout 步骤里加 token 参数（用 PAT 作为 submodule 鉴权），见[官方文档](https://github.com/actions/checkout#usage)。

## 第 5 步：创建首页 `content/index.md`

> [!warning] 这一步必须做，否则首页会显示成 XML
> 如果 `content/` 根目录没有 `index.md`，Quartz 不会生成 `index.html`，访问站点根路径会回退到 RSS feed（显示成一段 XML）。这是这套架构最常见的"踩坑第一站"。

`content/index.md` 是框架仓自有的首页模板（不进笔记仓），自定义站点首页：

```markdown
---
title: <你的站点名>
---

这里是<你的名字>的个人博客，记录<一句话简介>。

内容由 [Obsidian 笔记仓](https://github.com/<你的用户名>/<笔记仓>) 驱动，通过 GitHub Actions 自动构建部署——写完笔记 push，博客即自动更新。

## 内容主题

笔记主要围绕以下几个方向：

- **主题 A** — 简要描述
- **主题 B** — 简要描述

## 如何浏览

- 顶部搜索框支持全文搜索
- 左侧文件树按目录浏览
- 右侧图谱展示笔记间的关联
- 每篇笔记底部有反向链接，追溯引用关系

> 本博客基于 [Quartz v5](https://quartz.jzhao.xyz/) 搭建。
```

提交这个文件时，确认它真的进了版本控制（被 `.gitignore` 放行）：

```bash
git add content/index.md
git ls-files content/index.md   # 应该输出文件名，空输出说明没入库
```

## 第 6 步：配置站点 `quartz.config.yaml`

第 2 步跑 `npx quartz create` 时已经生成了一份 `quartz.config.yaml`，里面**完整包含**插件清单、layout 编排、默认配置。这一步只需要改 `configuration:` 段的几个字段——**不要手删 `plugins:` 和 `layout:` 段**，否则会破坏构建。

打开 `quartz.config.yaml`，只修改 `configuration:` 段下面这几个字段：

```yaml
configuration:
  pageTitle: <你的站点名>               # 浏览器标签和站点顶部显示的标题
  pageTitleSuffix: ""                   # 标题后缀，如 " · 你的名字"，可留空
  baseUrl: <你的域名>                    # 必须和第 2 步的 --baseUrl 一致
  locale: zh-CN                         # 界面语言（中文用 zh-CN，英文用 en-us）
  ignorePatterns:                       # 不发布到博客的文件/目录（跨层匹配，自动应用到 content/notes/ 下）
    - private
    - templates
    - .obsidian
    - .smtcmp*                          # Obsidian smart-composer 插件缓存
    - README.md                         # 仓库说明文件不发布
    - README.en.md
    - LICENSE*
    - folder-alias.json                 # Obsidian 插件配置文件
    - private-folder-alias.json         # 同上（私密版）
```

> [!warning] ignorePatterns 必须配齐
> 上面 9 条是"Obsidian vault 里有但不该发布"的常见文件。如果你的 vault 用了其他插件（会生成额外配置文件），把它们也加进来。`ignorePatterns` 跨层匹配，会自动应用到 `content/notes/` 子目录，所以笔记仓根目录的 README/LICENSE 也会被排除，不用单独配。

> [!tip] 想要个性化（主题色 / 字体 / 统计 / 加密页等）
> 默认配置是中性的（Quartz 默认主题、无统计）。如果想要自定义主题色板、字体、Plausible 统计、加密页等功能，参考[附录 B](#附录-b个性化定制说明可选)。完整配置样本见[附录 A](#附录-a完整-quartzconfigyaml-参考样本)。

改完后，确认 `quartz.lock.json` 也存在（第 2 步生成，锁定了社区插件版本，CI 按此安装）：

```bash
ls quartz.lock.json
# 输出文件名 = OK；不存在就重新跑 npx quartz create
```

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
      #    必须完整拉取（--unshallow）：submodule 默认是浅克隆，
      #    下一步还原笔记修改时间需要完整 git 历史
      - name: Update notes submodule to latest
        run: |
          git -C content/notes fetch --unshallow origin master 2>/dev/null \
            || git -C content/notes fetch origin master
          git -C content/notes checkout origin/master

      # 3. 按笔记的 git 提交时间还原文件 mtime
      #    actions/checkout 会把所有文件 mtime 重置为 checkout 时刻，
      #    导致博客上所有笔记的修改时间都显示成"今天"。
      #    这里遍历每个文件，用它的 git 最后提交时间覆盖回去。
      #
      #    为什么需要这一步？created-modified-date 插件的优先级是
      #    frontmatter > git > filesystem。正常情况插件读 submodule
      #    自己的 .git 就能拿到时间，但 Quartz 构建进程在框架仓根目录
      #    运行，插件读的是框架仓的 .git（而非 submodule 的），
      #    git 源对笔记失效 → fallback 到 filesystem mtime → 全显示今天。
      #    这一步直接把 filesystem mtime 修正回来，作为兜底。
      - name: Restore notes mtime from git history
        run: |
          cd content/notes
          git ls-files -z | while IFS= read -r -d '' f; do
            mtime=$(git log -1 --format='%ct' "$f" 2>/dev/null)
            if [ -n "$mtime" ]; then
              touch -d "@$mtime" "$f"
            fi
          done

      # 4. 构建
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

      # 5. 写入自定义域名
      #    有自定义域名：把 <你的域名> 换成实际域名（如 blog.example.com），保留这行
      #    无自定义域名：删掉这行（用 <用户名>.github.io/<框架仓> 默认地址）
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

> [!note] 为什么 token 给框架仓权限，却存在笔记仓？
> 这个 token 代表的是**对框架仓的 dispatch 权限**，但它是**在笔记仓的 workflow 里被消费**的（trigger workflow 在笔记仓跑）。所以：
> - **权限**指向框架仓（9.1 选框架仓）
> - **Secret** 存笔记仓（这里，因为 trigger workflow 在笔记仓执行）
> - 触发链路：笔记仓 push → 笔记仓 workflow 用这个 secret → 调框架仓的 dispatch API → 框架仓开始构建

### 9.3 在笔记仓添加 trigger workflow

在笔记仓创建 `.github/workflows/trigger-site.yml`：

```yaml
name: 触发博客重建

# 笔记仓内容变更后，通知框架仓重新构建部署博客
# 实现近实时联动：push 笔记 → 博客 1-2 分钟内自动更新
on:
  push:
    branches: [master]

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - name: 通知框架仓重建
        run: |
          set -e
          RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
            -H "Accept: application/vnd.github+json" \
            -H "Authorization: Bearer ${{ secrets.BLOG_SITE_PAT }}" \
            https://api.github.com/repos/<你的用户名>/<框架仓>/dispatches \
            -d '{"event_type":"notes-updated"}')
          echo "Dispatch response: $RESPONSE"
          if [ "$RESPONSE" != "204" ]; then
            echo "::warning::触发博客重建失败（HTTP $RESPONSE）。请检查 BLOG_SITE_PAT secret 是否已配置且未过期。"
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
cd ~/你的/vault/路径          # 确保在笔记仓目录
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

# 2. deploy.yml 的 build job 末尾有 echo "<你的域名>" > public/CNAME 这行
#    确认你把 <你的域名> 换成了实际域名（如 blog.example.com）

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

# 首次或 quartz.lock.json 变动后，按锁文件安装社区插件
npx quartz plugin install --from-config

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
npx quartz plugin install --from-config   # 社区插件也可能变，按锁文件重装
npx quartz build --serve       # 本地验证

# 3. 验证通过后推送，触发部署
git push origin main
```

## 目录结构速查

```
<框架仓>/
├── quartz.config.yaml        # 站点配置（站点名/主题/插件清单/layout 编排）
├── quartz.lock.json          # 社区插件版本锁（CI 按此装，不拉 latest）
├── quartz.config.default.yaml # 上游默认配置模板（npx quartz create 的来源）
├── content/
│   ├── index.md              # 首页模板（框架仓自有，入库）
│   ├── .gitkeep              # 占位（上游 Quartz 用来保留 content/ 目录）
│   └── notes/                # 笔记仓 submodule（不入主仓历史）
├── .gitmodules               # submodule 配置（指向笔记仓）
├── .gitignore                # 忽略 node_modules/public/.quartz/ 等
├── .github/workflows/
│   └── deploy.yml            # 构建 + 部署工作流
├── quartz/                   # Quartz v5 核心（上游源码，勿改）
└── docs/                     # Quartz 官方文档源（上游文件，可选保留）
```

| 分支 | 用途 |
|---|---|
| `main`（默认） | 业务层，日常提交，CI 从这构建 |
| `v5` | 上游镜像层，只接收 Quartz 更新 |

## 附录 A：完整 `quartz.config.yaml` 参考样本

> [!info] 怎么用这份附录
> 这是本站实际使用的 `quartz.config.yaml` 完整内容，**包含个性化定制**（主题色板、字体、统计、加密页等）。你按第 6 步跑 `npx quartz create` 生成的版本会和这里不同——**以你生成的为准**，这份只作参考，让你对照"哪些是默认、哪些是定制"。
>
> 定制项的说明见[附录 B](#附录-b个性化定制说明可选)。`configuration` 段几个易混字段：
> - `enableSPA: true` — 站点内链接用前端路由跳转（不刷新整页），体验更顺
> - `enablePopovers: true` — 鼠标悬停 `[[内部链接]]` 时弹出预览卡片
> - `enablePopovers` 等字段都是 `npx quartz create` 默认开启的，无需手动改

```yaml
configuration:
  pageTitle: 鸿影的博客
  pageTitleSuffix: " · 鸿影"
  enableSPA: true
  enablePopovers: true
  analytics:
    provider: plausible
  locale: zh-CN
  baseUrl: blog.honlnk.com
  ignorePatterns:
    - private
    - templates
    - .obsidian
    - .smtcmp*
    - README.md
    - README.en.md
    - LICENSE*
    - folder-alias.json
    - private-folder-alias.json
  theme:
    fontOrigin: googleFonts
    cdnCaching: true
    typography:
      header: Schibsted Grotesk
      body: Source Sans Pro
      code: IBM Plex Mono
    colors:
      lightMode:
        light: "#faf8f8"
        lightgray: "#e5e5e5"
        gray: "#b8b8b8"
        darkgray: "#4e4e4e"
        dark: "#2b2b2b"
        secondary: "#284b63"
        tertiary: "#84a59d"
        highlight: rgba(143, 159, 169, 0.15)
        textHighlight: "#fff23688"
      darkMode:
        light: "#161618"
        lightgray: "#393639"
        gray: "#646464"
        darkgray: "#d4d4d4"
        dark: "#ebebec"
        secondary: "#7b97aa"
        tertiary: "#84a59d"
        highlight: rgba(143, 159, 169, 0.15)
        textHighlight: "#b3aa0288"
plugins:
  - source: github:quartz-community/created-modified-date
    enabled: true
    options:
      defaultDateType: modified
      priority:
        - frontmatter
        - git
        - filesystem
    order: 10
  - source: github:quartz-community/syntax-highlighting
    enabled: true
    options:
      theme:
        light: github-light
        dark: github-dark
      keepBackground: false
    order: 20
  - source: github:quartz-community/obsidian-flavored-markdown
    enabled: true
    options:
      enableInHtmlEmbed: false
      enableCheckbox: true
    order: 30
  - source: github:quartz-community/github-flavored-markdown
    enabled: true
    order: 40
  - source: github:quartz-community/table-of-contents
    enabled: true
    order: 50
    layout:
      position: right
      priority: 30
  - source: github:quartz-community/crawl-links
    enabled: true
    options:
      markdownLinkResolution: shortest
    order: 60
  - source: github:quartz-community/description
    enabled: true
    order: 70
  - source: github:quartz-community/latex
    enabled: true
    options:
      renderEngine: katex
    order: 80
  - source: github:quartz-community/citations
    enabled: false
    order: 85
  - source: github:quartz-community/hard-line-breaks
    enabled: false
    order: 90
  - source: github:quartz-community/ox-hugo
    enabled: false
    order: 91
  - source: github:quartz-community/roam
    enabled: false
    order: 92
  - source: github:quartz-community/fonts
    enabled: true
  - source: github:quartz-community/remove-draft
    enabled: true
  - source: github:quartz-community/explicit-publish
    enabled: false
  - source: github:quartz-community/unlisted-pages
    enabled: true
    options: {}
    order: 45
  - source: github:quartz-community/encrypted-pages
    enabled: true
    options:
      iterations: 600000
      passwordField: password
      unlistWhenEncrypted: false
      outputPath: static/encryptedContentIndex.json
  - source: github:quartz-community/stacked-pages
    enabled: false
    layout:
      position: afterBody
      priority: 50
      display: all
  - source: github:quartz-community/alias-redirects
    enabled: true
  - source: github:quartz-community/content-index
    enabled: true
    options:
      enableSiteMap: true
      enableRSS: true
  - source: github:quartz-community/favicon
    enabled: true
  - source: github:quartz-community/og-image
    enabled: true
  - source: github:quartz-community/cname
    enabled: true
  - source: github:quartz-community/canvas-page
    enabled: true
  - source: github:quartz-community/content-page
    enabled: true
  - source: github:quartz-community/folder-page
    enabled: true
  - source: github:quartz-community/tag-page
    enabled: true
  - source: github:quartz-community/explorer
    enabled: true
    layout:
      position: left
      priority: 50
  - source: github:quartz-community/graph
    enabled: true
    layout:
      position: right
      priority: 10
  - source: github:quartz-community/search
    enabled: true
    layout:
      position: left
      priority: 20
      group: toolbar
      groupOptions:
        grow: true
  - source: github:quartz-community/backlinks
    enabled: true
    layout:
      position: right
      priority: 50
  - source: github:quartz-community/article-title
    enabled: true
    layout:
      position: beforeBody
      priority: 10
  - source: github:quartz-community/content-meta
    enabled: true
    layout:
      position: beforeBody
      priority: 20
  - source: github:quartz-community/tag-list
    enabled: false
    layout:
      position: beforeBody
      priority: 30
  - source: github:quartz-community/page-title
    enabled: true
    layout:
      position: left
      priority: 10
  - source: github:quartz-community/darkmode
    enabled: true
    layout:
      position: left
      priority: 30
      group: toolbar
  - source: github:quartz-community/reader-mode
    enabled: true
    layout:
      position: left
      priority: 35
      group: toolbar
  - source: github:quartz-community/breadcrumbs
    enabled: true
    layout:
      position: beforeBody
      priority: 5
      condition: not-index
  - source: github:quartz-community/comments
    enabled: false
    options:
      provider: giscus
      options: {}
    layout:
      position: afterBody
      priority: 10
  - source: github:quartz-community/footer
    enabled: true
    options:
      links:
        GitHub: https://github.com/jackyzha0/quartz
        Discord Community: https://discord.gg/cRFFHYye7t
  - source: github:quartz-community/recent-notes
    enabled: false
  - source: github:quartz-community/spacer
    enabled: true
    options: {}
    order: 25
    layout:
      position: left
      priority: 25
      display: mobile-only
  - source: github:quartz-community/bases-page
    enabled: true
    options: {}
    order: 50
  - source: github:quartz-community/note-properties
    enabled: true
    options:
      includeAll: false
      includedProperties:
        - description
        - tags
        - aliases
      excludedProperties: []
      hidePropertiesView: false
      delimiters: ---
      language: yaml
    order: 5
    layout:
      position: beforeBody
      priority: 15
      display: all
layout:
  groups:
    toolbar:
      priority: 35
      direction: row
      gap: 0.5rem
  byPageType:
    "404":
      positions:
        beforeBody: []
        left: []
        right: []
    content: {}
    folder:
      exclude:
        - reader-mode
      positions:
        right: []
    tag:
      exclude:
        - reader-mode
      positions:
        right: []
    canvas: {}
    bases: {}
```

## 附录 B：个性化定制说明（可选）

附录 A 里这些是本站实际采用的定制，按需选用。

### B.1 主题色板 + 字体

在 `configuration.theme` 段。本站用 Google Fonts 加载三种字体，并自定义了完整 light/dark 调色板：

- `fontOrigin: googleFonts` + `cdnCaching: true` — 从 Google Fonts CDN 加载并缓存
- `typography` — 标题用 `Schibsted Grotesk`、正文用 `Source Sans Pro`、代码用 `IBM Plex Mono`
- `colors.lightMode` / `colors.darkMode` — 9 种语义色（light/lightgray/gray/darkgray/dark/secondary/tertiary/highlight/textHighlight），全用十六进制或 rgba

> [!tip] 想换主题色
> 把 `secondary` 和 `tertiary` 改成你的主色和强调色即可（这两个影响链接、按钮、高亮），其余保持不变。配色生成可参考 [Realtime Colors](https://www.realtimecolors.com/)。

### B.2 Plausible 统计

```yaml
analytics:
  provider: plausible
```

需要先在 [Plausible](https://plausible.io/)（或自托管实例）建站点，并在 `quartz.config.yaml` 同级按 Plausible 文档配置 script 来源。详细参数见 Quartz 官方 [analytics 文档](https://quartz.jzhao.xyz/features/analytics)。不开统计就删掉 `analytics` 段。

### B.3 加密页功能（`encrypted-pages`）

```yaml
- source: github:quartz-community/encrypted-pages
  enabled: true
  options:
    iterations: 600000
    passwordField: password
    unlistWhenEncrypted: false
    outputPath: static/encryptedContentIndex.json
```

让带 `password` frontmatter 的笔记变成访问需密码的页面。`iterations` 是 PBKDF2 迭代次数（越高越安全越慢），`outputPath` 是加密索引输出路径。不需要这个功能就把 `enabled` 改 false。

### B.4 阅读模式 / 面包屑 / 笔记属性

几个体验增强插件，按需开启：

| 插件 | 作用 |
|---|---|
| `reader-mode` | 笔记页右上角加"阅读模式"按钮，隐藏侧栏 |
| `breadcrumbs` | 笔记顶部显示目录层级路径（`首页 / 目录 / 当前页`） |
| `note-properties` | 在笔记正文上方展示指定的 frontmatter 字段（本站显 `description/tags/aliases`） |
| `unlisted-pages` | 带 `unlisted: true` 的笔记不进文件树/RSS，但可直接访问 |
| `remove-draft` | 带 `draft: true` 的笔记完全不发布 |

启用就在 `plugins:` 列表里加 `enabled: true`，按附录 A 的 `layout` 字段配置位置。

### B.5 标题后缀

```yaml
pageTitleSuffix: " · 鸿影"
```

浏览器标签页和 `<title>` 显示成 `笔记名 · 鸿影`。改成你自己的署名或留空字符串。

### B.6 layout 编排

`layout:` 段控制哪些插件出现在页面哪个位置。本站的定制：

- **toolbar 组**（`groups.toolbar`）：把 search、darkmode、reader-mode 横向排在顶部工具栏（priority 35，row 布局）
- **folder/tag 页精简**（`byPageType.folder` / `tag`）：文件夹页和标签页隐藏 reader-mode 和右侧栏（只保留主内容）
- **404 页极简**：所有位置清空

> [!note] layout 的配置哲学
> Quartz 的 layout 系统是"插件 × 位置 × 页面类型"的三维编排，每个插件有自己的 `position`（left/right/beforeBody/afterBody/toolbar）和 `priority`（同位置内排序）。完整规则见官方 [layout 文档](https://quartz.jzhao.xyz/features/layout)。**一般不用动 layout 段**，`npx quartz create` 生成的默认编排已经够用。

## 附录 C：完整 `.gitignore`

Quartz clone 下来自带一份默认 `.gitignore`，一般够用。如果发现构建产物或编辑器缓存被误提交，对照下面这份（本站实际使用的完整版）补齐缺失项：

```gitignore
.DS_Store
.gitignore
node_modules
public
prof
tsconfig.tsbuildinfo
.obsidian
.quartz-cache
private/
.replit
replit.nix
.quartz/
quartz/cli/tui/dist/

# content/notes/ 通过 git submodule 引用笔记仓
# submodule 只在主仓记录 commit 指针，笔记内容不入主仓历史
# （注意：不能用 .gitignore 忽略 content/notes，否则 Quartz 的 gitignore:true 会跳过构建）
# content/index.md 是框架仓自有的首页模板，入库管理
```

> [!note] 为什么 `.gitignore` 里有一行 `.gitignore`？
> 这是上游 Quartz 的设计，让 `.gitignore` 自身也不入库（保持本地配置）。如果你希望 `.gitignore` 入库被团队共享，删掉这行即可。

> [!warning] 不要加 `content/notes` 到 .gitignore
> Quartz 构建时默认遵守 `.gitignore`（`gitignore: true`）。如果把 `content/notes/` 加进忽略列表，**所有笔记会从构建中消失**。submodule 已经让笔记内容不入主仓历史，不需要再 gitignore。

## 相关笔记

- [[blog-publish-with-quartz-submodule]] — 完整实现方案（含踩坑记录和设计决策）
- [[git-submodules-nested-repositories-guide]] — git submodule 深入指南
- [[obsidian-quickstart]] — Obsidian 快速上手

## 参考链接

- [Quartz v5 官方文档](https://quartz.jzhao.xyz/)
- [GitHub Actions: repository_dispatch](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows#repository_dispatch)
- [Fine-grained PAT 权限要求](https://docs.github.com/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens)
- [GitHub Pages 部署](https://docs.github.com/pages)
