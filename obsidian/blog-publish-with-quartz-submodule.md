---
title: Obsidian 笔记自动发布为博客的实现方案（Quartz + git submodule）
date: 2026-07-18
tags:
  - obsidian
  - quartz
  - blog
  - git-submodule
  - github-actions
  - ci-cd
status: completed
---

# Obsidian 笔记自动发布为博客的实现方案（Quartz + git submodule）

> [!info] 方案信息
> - **上线地址**：[blog.honlnk.com](https://blog.honlnk.com)
> - **框架**：[Quartz v5](https://quartz.jzhao.xyz/)
> - **核心机制**：Obsidian 笔记仓 → git submodule → Quartz 构建 → GitHub Pages
> - **最终效果**：在 Obsidian 写笔记 push 后，博客 1-2 分钟内自动更新

## 背景与目标

我有一个 Obsidian vault（`honlnk-blog` 仓），日常在里面记笔记。希望这些笔记能自动发布成一个带样式、支持 wikilink/反向链接/图谱/全文搜索的静态博客，且满足：

1. **笔记仓保持纯净**——它就是 Obsidian vault，不能被博客框架代码污染
2. **框架可跟随上游更新**——Quartz 升级时零冲突
3. **写完即发**——push 笔记后博客自动更新，无需手动操作

经过几次迭代（踩了不少坑），最终用 **Quartz + git submodule** 的方案落地。本文记录完整实现，供日后参考或重建时复用。

## 整体架构

```
┌─────────────────┐         ┌──────────────────────────────────┐
│  honlnk-blog    │  submodule  │  honlnk-blog-site（框架仓）      │
│  （笔记仓）      │◄───────────│                                  │
│                 │            │  ├── quartz/        上游框架     │
│  Obsidian vault │            │  ├── content/index.md 首页模板   │
│  *.md / *.canvas│            │  ├── content/notes/ → 笔记仓     │
└─────────────────┘            │  ├── quartz.config.yaml          │
        │                      │  └── .github/workflows/deploy.yml│
        │ push                 └──────────────────────────────────┘
        ▼                              │
 trigger-site.yml                     │ repository_dispatch
 （发 dispatch 通知）─────────────────►│
                                      ▼
                              deploy.yml 触发：
                              1. checkout 框架仓 + submodule
                              2. submodule update 到 master 最新
                              3. quartz build
                              4. 部署 GitHub Pages
                                      │
                                      ▼
                              blog.honlnk.com 自动更新
```

**两个仓库的分工**：

| 仓库 | 角色 | 内容 |
|---|---|---|
| `honlnk/honlnk-blog` | 数据源（笔记仓） | Obsidian vault，只管写笔记 |
| `honlnk/honlnk-blog-site` | 框架仓（本仓） | Quartz 配置、首页模板、CI 部署 |

## 分支拓扑：main（业务层）+ v5（上游镜像层）

框架仓采用**双分支分层**设计，隔离业务定制与上游代码：

```
upstream/v5 (Quartz 官方)
    │  纯净上游，含官方 workflow（ci/build-preview/docker 等）
    │
    │  git fetch upstream && git merge upstream/v5
    ▼
origin/v5 (上游镜像层 / 缓冲区)
    │  和 upstream/v5 保持 0 diff
    │  作用：接收上游更新的缓冲层，升级冲突在这里解决
    │
    │  git checkout main && git merge v5
    ▼
origin/main (业务层，默认分支)
    │  所有自有定制：deploy.yml、首页模板、文档、submodule
    │  CI 从这里构建部署
    └─→ blog.honlnk.com
```

**为什么要分两层**：

- 直接在 `v5` 上做业务，会和上游同名分支语义混淆（`upstream/v5` 是 Quartz 版本线，不是业务分支）
- 上游出 v6 时，`v5` 这个名字会变成历史遗留
- 两层缓冲让"上游同步"和"业务开发"彻底解耦——冲突在 v5 解决，不污染 main 的历史

**两个分支的职责**：

| 分支 | 作用 | 谁在上面写 |
|---|---|---|
| `v5` | 上游镜像，保持和 `upstream/v5` 完全一致 | 只接收 upstream 合并，**不直接写业务** |
| `main` | 业务层，默认分支，CI 构建 | 所有自有定制都在这里提交 |

> [!warning] 别在 v5 上直接做业务改动
> v5 的职责是"纯净上游镜像"。如果在 v5 上直接写业务，下次 `merge upstream/v5` 时会和你的改动冲突，违背"缓冲层"的设计意图。业务改动一律在 main 上做。

## 关键设计决策与踩过的坑

这一节是全文重点——记录几个看似可行实则踩坑的方案，以及为什么最终选 submodule。

### 坑 1：`rm -rf content` + checkout 笔记仓

> [!warning] 最初方案（已废弃）
> ```yaml
> - run: rm -rf content
> - uses: actions/checkout@v7
>   with:
>     repository: honlnk/honlnk-blog
>     path: content
> ```

**思路**：清空 `content/`，把笔记仓整个 checkout 进去。

**问题**：
- 笔记仓根目录没有 `index.md`，Quartz 不会生成首页 HTML
- 访问 `blog.honlnk.com/` 返回的是 RSS XML（`index.xml`），博客首页直接报废
- `rm -rf` 是暴力覆盖，框架仓自有的 `content/index.md` 也会被删

**教训**：不要用"清空 + 覆盖"的方式注入外部内容，应该让框架资产和外部内容和平共处。

### 坑 2：用 `.gitignore` 忽略 `content/notes/`

> [!warning] 看似合理的方案（已废弃）
> ```gitignore
> content/*
> !content/index.md
> ```
> 笔记 checkout 到 `content/notes/`，用 `.gitignore` 忽略，不入库。

**问题**：Quartz 的 glob 配置启用了 `gitignore: true`（见 `quartz/util/glob.ts`）：

```ts
await globby(pattern, {
  cwd,
  ignore: ignorePatterns,
  gitignore: true,  // ← 会遵守所有 .gitignore 规则
})
```

这意味着**被 `.gitignore` 忽略的文件，Quartz 构建时也会跳过**。结果：

- 加 `.gitignore` 忽略 `content/notes/` → Quartz 构建只扫到 1 个文件（首页），**57 篇笔记全部消失**
- 不忽略 → 笔记正常构建，但会被 `git add` 进框架仓，污染历史

这是个**两难矛盾**，普通目录方案无法同时满足"不入库"和"能构建"。

### 坑 3：submodule 笔记的修改时间全变成"今天"

> [!warning] 现象
> 上线后发现**所有笔记的修改时间都显示为今天**（CI 运行那天），包括一年前写的老笔记。排序和日期全部失效。

**根因有两层，叠加导致**：

**第 1 层：submodule 默认浅克隆**

`git submodule add` 默认只拉**浅层历史**（几个 commit），不是完整历史。即使 `git fetch origin`，也只会显示"up to date"——因为 ref 是最新的，但 commit 对象不完整，`git log` 走到断点就停。

```
远端 honlnk/honlnk-blog：    94 个 commit（完整）
submodule 本地 content/notes： 只有 4 个 commit（浅克隆）
```

**第 2 层：`actions/checkout` 重置 mtime**

即使 submodule 历史完整，`actions/checkout` 检出文件时会把**所有文件的 mtime 重置为 checkout 那一刻**（CI 运行时），而不是 git 历史里的真实时间。

**第 3 层：Quartz 插件优先读 filesystem mtime**

查看 `created-modified-date` 插件源码（`.quartz/plugins/created-modified-date/src/transformer.ts`）：

```ts
for (const source of opts.priority) {
  if (source === "filesystem") {
    modified ||= st.mtimeMs;                    // ← 优先读文件系统 mtime
  } else if (source === "git" && repo) {
    modified ||= await repo.getFileLatestModifiedDateAsync(...);  // 最后才读 git
  }
}
```

即使配置 `priority: [frontmatter, git, filesystem]`，`git` 来源也拿不到正确时间——因为插件用 `Repository.discover(ctx.argv.directory)` 定位 git 仓库，`ctx.argv.directory` 是 `content`（框架仓内容目录），往上找到的是**框架仓的 `.git`**，而不是 submodule 的 `.git`。框架仓不跟踪 submodule 内部文件，查询失败，最终回退到 filesystem（被重置过的 mtime）。

**排查过程的弯路（记录于此警示未来）**：

| 诊断轮次 | 假设 | 结论 |
|---|---|---|
| 第 1 轮 | `--depth 1` fetch 太浅 | ❌ 错误，已 revert |
| 第 2 轮 | 远端历史被误操作丢失 | ❌ 虚惊，远端 94 commit 完整 |
| 第 3 轮 | **浅克隆 + mtime 重置 + 插件读 git 失败** | ✅ 真根因 |

> [!danger] 诊断教训
> 看到 submodule 只有 4 个 commit 时，一度以为是自己之前 cherry-pick 误操作导致远端历史丢失，差点强推恢复。**幸好多查一步**——用 GitHub API 直接核实远端 commit 数（94 个），才发现远端是完整的，问题只在 submodule 本地。**强推前必须核实"远端真的缺吗"**，否则反而可能造成真正的事故。

**修复方案**：在 CI 构建前增加两步（见 `deploy.yml`）：

```yaml
# 1. 完整拉取 submodule 历史（unshallow）
- name: Update notes submodule to latest
  run: |
    git -C content/notes fetch --unshallow origin master 2>/dev/null \
      || git -C content/notes fetch origin master
    git -C content/notes checkout origin/master

# 2. 用 git 真实时间修复 mtime（绕过插件读 git 失败的问题）
- name: Restore notes mtime from git history
  run: |
    cd content/notes
    git ls-files -z | while IFS= read -r -d '' f; do
      mtime=$(git log -1 --format='%ct' "$f" 2>/dev/null)
      if [ -n "$mtime" ]; then
        touch -d "@$mtime" "$f"     # Linux touch 支持 -d @timestamp
      fi
    done
```

第 2 步是关键——既然插件读不到 submodule 的 git 时间，就直接把 git 时间**写进文件 mtime**，让插件从 filesystem 来源也能拿到真实值。`touch` 用 Linux 语法（CI 是 Ubuntu），`@timestamp` 是 unix 时间戳格式。

### 最终方案：git submodule

> [!success] 采用的方案
> 把笔记仓作为 **git submodule** 挂载到 `content/notes/`。

**为什么 submodule 能解决矛盾**：

| 需求 | submodule 如何满足 |
|---|---|
| 笔记不入框架仓历史 | submodule 只在主仓记录一个 commit 指针，内容不进主仓 git tree |
| Quartz 能扫描到笔记 | submodule 是**独立的 git 仓库**（有自己的 `.git`），主仓的 `.gitignore` 规则**不会穿透**进 submodule |
| 本地预览零配置 | `git submodule update --init` 即可，不用手动 clone |
| CI 拿最新笔记 | `git -C content/notes checkout origin/master` 一行命令 |

关键点在于 globby 的 `gitignore: true` 行为：它扫描到 submodule 内部时，会以 submodule 的根为新的 git 仓库边界，主仓的忽略规则失效。这是 git submodule 的固有特性，正好契合需求。

## 完整实现步骤

### 第 1 步：基于 Quartz v5 初始化框架仓

```bash
git clone https://github.com/jackyzha0/quartz.git honlnk-blog-site
cd honlnk-blog-site

# upstream 保留指向 Quartz 官方（方便升级），origin 指向自己的仓库
git remote rename origin upstream
git remote add origin git@github.com:honlnk/honlnk-blog-site.git

npm i
npx quartz create --template default --strategy new \
  --baseUrl blog.honlnk.com --links shortest
```

然后建立 main/v5 双分支拓扑（详见上一节"分支拓扑"）：

```bash
# 当前在 v5 分支（从上游 clone 来的）
# 1. 把 v5 改名为 main（业务层，保留所有改动历史）
git branch -m v5 main
git push -u origin main

# 2. 从 upstream 重新建一个纯净的 v5 分支（上游镜像层）
git branch v5 upstream/v5
git push -u origin v5

# 3. 把 GitHub 默认分支改成 main
gh repo edit honlnk/honlnk-blog-site --default-branch main

# 4. 回到 main 做后续所有业务改动
git checkout main
```

> [!tip] upstream 的价值
> 保留 upstream 让后续 `npx quartz update` 能拉取上游更新。关键是**所有自有定制只做"覆盖/新增"，不删改上游源码**，这样升级时零合并冲突。

### 第 2 步：配置 `quartz.config.yaml`

关键字段：

```yaml
configuration:
  pageTitle: 鸿影的博客
  pageTitleSuffix: " · 鸿影"
  baseUrl: blog.honlnk.com
  locale: zh-CN
  ignorePatterns:
    - private               # Obsidian 私有目录
    - templates             # 模板目录
    - .obsidian             # Obsidian 配置
    - .smtcmp*              # smart-composer 缓存
    - README.md             # 仓库说明文件不发布
    - README.en.md
    - LICENSE*
    - folder-alias.json     # smart-composer 配置
```

> [!note] ignorePatterns 的跨层特性
> `ignorePatterns` 是 fast-glob 语法，会**跨层匹配**——写在根的 `README.md` 规则会自动应用到 `content/notes/` 子目录下所有同名文件。所以笔记仓根目录的 README/LICENSE 会被正确排除，无需为子目录额外配置。

### 第 3 步：添加笔记仓为 submodule

```bash
git submodule add https://github.com/honlnk/honlnk-blog.git content/notes
```

生成的 `.gitmodules`：

```ini
[submodule "content/notes"]
    path = content/notes
    url = https://github.com/honlnk/honlnk-blog.git
```

### 第 4 步：创建首页模板 `content/index.md`

Quartz 默认会为 `content/` 根目录生成一个 folder listing 页，但样式简陋。写一个 `content/index.md` 可以自定义首页：

```markdown
---
title: 鸿影的博客
---

这里是鸿影的个人博客，记录我在软件开发过程中的实践与思考。

## 内容主题
- **AI** — LLM 应用、智能体
- **开发工具** — Git、pnpm、Obsidian
- ...
```

> [!important] index.md 的归属
> `content/index.md` 是**框架仓的展示层资产**，入库管理。它不属于笔记内容，不随笔记仓更新。这样首页样式与笔记内容彻底解耦。

### 第 5 步：配置 `.gitignore`

```gitignore
# 关键：不要用 content/* 忽略整个 content 目录
# 否则会连带忽略 content/notes submodule 和 content/index.md
# submodule 自身需要被 git 跟踪（记录 commit 指针）
```

注意：**不要在 `.gitignore` 里写 `content/*` 或 `content/notes/`**。submodule 需要被主仓跟踪（只跟踪指针，不跟踪内容），如果忽略了会导致 `git submodule add` 失败。

### 第 6 步：编写 CI 部署工作流

`.github/workflows/deploy.yml` 核心逻辑：

```yaml
name: Deploy blog to GitHub Pages

on:
  push:
    branches: [v5]                          # 框架仓自身改动触发
  repository_dispatch:
    types: [notes-updated]                  # 笔记仓 push 联动触发
  workflow_dispatch:                        # 手动触发

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # 1. checkout 框架仓 + 初始化 submodule
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
          submodules: recursive             # 初始化 content/notes submodule

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

      # 4. 写入自定义域名
      - run: echo "blog.honlnk.com" > public/CNAME
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

### 第 7 步：配置笔记仓的联动 trigger

笔记仓 `.github/workflows/trigger-site.yml`：push 后通知框架仓重建。

```yaml
name: 触发博客重建

on:
  push:
    branches: [master]

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - name: 通知 honlnk-blog-site 重建
        run: |
          RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
            -H "Authorization: Bearer ${{ secrets.BLOG_SITE_PAT }}" \
            -H "Accept: application/vnd.github+json" \
            https://api.github.com/repos/honlnk/honlnk-blog-site/dispatches \
            -d '{"event_type":"notes-updated"}')
          if [ "$RESPONSE" != "204" ]; then
            echo "::error::触发失败 HTTP $RESPONSE"
            exit 1
          fi
```

> [!warning] PAT 权限的关键坑
> 触发 `repository_dispatch` 需要的权限是 **`Contents: Read and write`**，**不是** `Actions`。
>
> 这是真实踩过的坑：按直觉给 Actions 权限（甚至给了 Read and write），dispatch 都返回 HTTP 403。查 GitHub 官方文档和社区讨论才确认：`/repos/{owner}/{repo}/dispatches` 端点归属 Contents 权限域。详见 [permissions-required-for-fine-grained-PAT](https://docs.github.com/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens)。
>
> PAT 创建后存为笔记仓的 secret `BLOG_SITE_PAT`。

## 本地预览

```bash
cd honlnk-blog-site
npm install

# 初始化 / 更新 submodule
git submodule update --init --recursive

# 想拉笔记仓最新（而非锁定的 commit）
git -C content/notes fetch origin master
git -C content/notes checkout origin/master

npx quartz build --serve   # → http://localhost:8080
```

## 升级 Quartz

升级走 **v5 → main 两层缓冲**，冲突在 v5 解决，不污染 main：

```bash
cd honlnk-blog-site

# 1. 先 fetch 上游最新
git fetch upstream

# 2. 在 v5（上游镜像层）合并 upstream/v5 的更新
git checkout v5
git merge upstream/v5
git push origin v5

# 3. 切到 main，把更新过的 v5 合进来
git checkout main
git merge v5
# 如有冲突，在 main 上解决（通常是配置文件的合并，很少）
npm install          # 依赖可能变了
npx quartz build --serve   # 本地验证构建无误

# 4. 验证通过后推送，触发部署
git push origin main
```

由于所有自有定制都是"覆盖/新增"（配置文件、首页模板、CI、文档），没有修改 `quartz/` 源码，升级时基本零冲突。

> [!note] 为什么要两层
> 如果直接在 main 上 `merge upstream/v5`，上游的官方 workflow（`ci.yaml`、`docker-build-push.yaml` 等）会和你的 `deploy.yml` 混在同一个分支。用 v5 当缓冲层，可以让 v5 保持"纯净上游"（含官方 workflow），main 只管业务，职责清晰隔离。

## 故障排查

### 首页显示 XML 而非 HTML

检查 `content/index.md` 是否存在且已入库。Quartz 没找到 `index` slug 的源文件时，`/` 会回退到 `index.xml`（RSS）。

### 笔记内容没出现在博客上

按数据流顺序排查：

1. **笔记仓 push 成功了吗** → 看 `honlnk/honlnk-blog/actions` 的 trigger workflow
2. **trigger 报 403** → `BLOG_SITE_PAT` 权限错（应为 Contents:write）或过期
3. **框架仓没触发** → 手动 `gh workflow run deploy.yml --repo honlnk/honlnk-blog-site`
4. **submodule 没更新** → 检查 deploy.yml 的 "Update notes submodule" 步骤日志
5. **构建报错** → `gh run view --log-failed --repo honlnk/honlnk-blog-site`

### 某篇笔记显示异常

| 现象 | 原因 |
|---|---|
| wikilink 断链 | 目标笔记未发布，或文件名不匹配 |
| 图片不显示 | 图片没在笔记仓内，或用了绝对路径 |
| 日期显示为今天 | 笔记未被 git 跟踪（本地新文件），push 后 CI 用真实 git 时间 |

### deploy 报 "Branch X is not allowed to deploy to github-pages"

改过默认分支名（比如 v5 → main）后，deploy 的 build 成功了但 deploy 步骤报：

```
Branch "main" is not allowed to deploy to github-pages due to environment protection rules.
```

原因：GitHub Pages 的 `github-pages` environment 有**分支白名单**，改默认分支后白名单不会自动更新。修复：

```bash
# 看当前允许哪些分支部署
gh api repos/honlnk/honlnk-blog-site/environments/github-pages/deployment-branch-policies \
  --jq '.branch_policies[] | .name'

# 加上新分支（main）
gh api --method POST repos/honlnk/honlnk-blog-site/environments/github-pages/deployment-branch-policies \
  -f name=main

# 删掉旧分支（v5）的策略（用上一步查到的 id）
gh api --method DELETE repos/honlnk/honlnk-blog-site/environments/github-pages/deployment-branch-policies/<ID>
```

> [!warning] 这是个隐蔽坑
> 改分支名时容易漏这一步——因为 build 能成功（代码没问题），只有 deploy 步骤才会暴露。报错信息里的 "environment protection rules" 不直接说"分支白名单"，要绕个弯才想到。

### submodule 指针长期落后笔记仓 master

**这是正常现象，不是 bug**。框架仓里的 `content/notes` submodule 锁定的是某个 commit，但 CI 每次构建都会用 `git checkout origin/master` 跳过锁，直接拉笔记仓最新——所以**只在笔记仓 push，博客照样自动更新**，不用动框架仓。

如果想让框架仓的 submodule 指针也保持最新（纯美观/整洁目的）：

```bash
cd honlnk-blog-site
git -C content/notes checkout master   # 切到最新
git add content/notes
git commit -m "chore: 同步笔记 submodule 指针"
git push
```

但**不做也完全不影响博客工作**。

## 目录结构总览

```
honlnk-blog-site/
├── quartz.config.yaml        # 站点配置
├── quartz.lock.json          # 插件版本锁
├── content/
│   ├── index.md              # 框架仓首页模板（入库）
│   ├── .gitkeep
│   └── notes/                # 笔记仓 submodule（指向 honlnk/honlnk-blog）
├── .gitmodules               # submodule 配置
├── .github/workflows/
│   └── deploy.yml            # 构建 + 部署（只在 main 分支生效）
├── quartz/                   # Quartz v5 核心（上游，勿改）
└── docs/project/             # 项目文档
```

**分支结构**：

| 分支 | 内容 | 用途 |
|---|---|---|
| `main`（默认） | 业务层：自定义配置、首页、CI、submodule | 日常提交，CI 从这构建部署 |
| `v5` | 纯净上游镜像，和 `upstream/v5` 保持一致 | 接收 Quartz 更新的缓冲层 |
| `upstream/v5` | Quartz 官方代码（remote 跟踪） | `git fetch upstream` 时更新 |



## 相关笔记

- [[open-source-online-notes]] — 开源笔记方案调研
- [[git-submodules-nested-repositories-guide]] — git submodule 深入指南
- [[obsidian-quickstart]] — Obsidian 快速上手

## 参考链接

- [Quartz v5 官方文档](https://quartz.jzhao.xyz/)
- [GitHub Actions: repository_dispatch](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows#repository_dispatch)
- [Fine-grained PAT 权限要求](https://docs.github.com/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens)
