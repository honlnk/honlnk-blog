---
tags:
  - AI工具
  - ZCode
  - WorkBuddy
  - 踩坑记录
  - 内容安全
aliases:
  - ZCode gitStatus 触发内容过滤
  - WorkBuddy content-filter 误报
date: 2026-08-08
status: published
type: 技术调研
---

# ZCode gitStatus 触发 WorkBuddy 内容过滤排查全记录

> [!abstract] 摘要
> 在 ZCode 中使用 WorkBuddy（腾讯 Copilot，`copilot.tencent.com`）作为模型 provider 时，所有 git 仓库项目发任意消息（哪怕只发"你好"）都会被拦截，返回"系统检测到您当前输入的信息存在敏感内容"。经过对 ZCode 日志、数据库、model-io 请求记录的逐层拆解，最终通过 curl 二分重放实验，精确定位到触发腾讯 WAF content-filter 的根因是 ZCode 运行时自动注入的一句 git 状态信息。

> [!info] 背景
> WorkBuddy 是腾讯出品的全场景 AI 办公工作台，也可作为 OpenAI 兼容接口在 ZCode 等 vibe coding 工具中使用。它在 `https://copilot.tencent.com/v2/chat/completions` 提供 GLM 系列模型的流式接口。

## 一、现象

在 ZCode 中配置了两个 GLM provider：

| Provider | baseURL | 模型 | 现象 |
|---|---|---|---|
| 智谱官方（`builtin:bigmodel-coding-plan`） | `open.bigmodel.cn` | `GLM-5.2` | ✅ 正常 |
| WorkBuddy（`c4c3de40-…`） | `copilot.tencent.com/v2` | `glm-5.2` | ❌ git 项目全被拦 |
| 朋友中转（`d377abe6-…`，520355） | `aii.520355.xyz/v1` | `glm-5.2` | ❌ 同样被拦（走 workbuddy 上游） |

关键现象：**同一个 API key、同一条链路**，在 `ZCodeProject`（空目录）里能正常对话，但在任何 git 仓库里发"你好"就被拦。跨 6 个不同项目、2 个不同模型（glm-5.2、kimi-k3）全部复现。

## 二、排查过程

### 2.1 排除 API key 和网络问题

> [!tip] 数据来源：ZCode SQLite 数据库
> `~/.zcode/cli/db/db.sqlite` 的 `part` 表存储了每个会话的完整请求/响应记录，包含 `finishReason` 和 `tokens` 统计。

从数据库提取所有被拦截请求的元数据：

| 指标 | ❌ 被拦会话 | ✅ 正常会话 |
|---|---|---|
| provider / model / key | 完全相同 | 完全相同 |
| HTTP 状态 | completed（成功送达） | completed |
| **finishReason** | `other` / `content-filter` | `stop` |
| **input tokens** | **0** | 3995 |
| **output tokens** | **0** | 119 |

`input/output tokens = 0` 是铁证：模型根本没处理请求，是网关层直接返回了拒答话术。这不是 key 失效、不是网络超时、不是配置错误。

### 2.2 找到实际发送的完整 prompt

> [!note] ZCode 的 model-io rollout 文件
> `~/.zcode/cli/rollout/model-io-sess_<session-id>.jsonl` 记录了每个会话**实际发给模型的完整请求体**（system prompt、messages、tools 全部都在里面）。这是排查 prompt 问题的终极数据源。

从被拦会话的 rollout 文件里提取出 ZCode 实际拼接的请求结构：

```
messages[0]  role=system  "You are ZCode, an interactive coding agent"
messages[1]  role=system  身份定义 + 安全规则 + Harness 规则（~1153字符）
messages[2]  role=system  编码风格 + Environment + gitStatus（~2077字符）  ← 关键
messages[3]  role=user    skills 列表（~4917字符）
messages[4]  role=user    <system-reminder> + 当前日期 + 上下文
messages[5]  role=user    "你好"
+ 33个 tools 定义
```

用户只打了"你好"两个字，但实际发给模型的是 **8000+ 字符的拼接 prompt**。

### 2.3 curl 二分重放实验

用被拦会话的完整请求体**原样重放**，成功复现 `content_filter`：

```
curl → copilot.tencent.com/v2/chat/completions
请求体: 完整 6 条 messages + 33 个 tools
结果: finish_reason=content_filter, content="抱歉，系统检测到…敏感内容…"
```

然后逐条 message 剔除测试，定位到 **system message[2]**（2077字符）单独就能触发。继续对 message[2] 按段落二分：

| 段落 | 结果 |
|---|---|
| 编码风格（"Write code that reads like…"） | ✅ 正常 |
| 行为准则（"For actions that are hard to reverse…"） | ✅ 正常 |
| 技能指引（"Session-specific guidance"） | ✅ 正常 |
| 环境信息（"# Environment"） | ✅ 正常 |
| 上下文管理（"# Context management"） | ✅ 正常 |
| **git 状态（"gitStatus: This is the git status…"）** | **❌ 被拦** |

继续对 gitStatus 段落内部逐行二分：

| 行 | 结果 |
|---|---|
| `gitStatus: This is the git status at the start...` | ✅ 正常 |
| `Current branch: master` | ✅ 正常 |
| **`Main branch (you will usually use this for PRs): master`** | **❌ 被拦** |
| `Git user: honlnk` | ✅ 正常 |
| `Status:\nM blog` | ✅ 正常 |
| `Recent commits:\n...` | ✅ 正常 |

**精确定位到一句话**：`Main branch (you will usually use this for PRs): master`。

### 2.4 句式拆解

对这句话进一步拆解，确定触发条件：

| 测试内容 | 结果 |
|---|---|
| `Main branch: master` | ✅ 正常 |
| `Main branch (use this for PRs): master` | ✅ 正常 |
| `Main branch (you will use this for PRs): master` | ✅ 正常 |
| **`Main branch (you will usually use this for PRs): master`** | **❌ 被拦** |
| `Main branch (you will usually use this for PRs): develop` | ❌ 被拦（跟分支名无关） |
| `Main branch (you will usually use this for PRs): main` | ❌ 被拦 |
| `Main branch (you will usually use this for commits): master` | ✅ 正常（跟 PRs 有关） |
| `Main branch (you will usually use this for reviews): master` | ✅ 正常 |
| 同样内容放 user 消息（非 system） | ✅ 正常（只拦 system 角色） |

> [!warning] 根因判定
> 腾讯 Copilot 的 WAF 内容安全引擎，对 **system 角色消息**中的 `"you will usually use this for PRs"` 表述做了模式匹配——这种"你将通常用它来做 PR"的指令式语句被判定为可疑的 prompt 注入/越狱模式，触发了 content-filter。这是一个**误报**，该句实际是 ZCode 注入的 git 上下文说明。

## 三、ZCode 的 gitStatus 注入机制

> [!info] 源码定位
> 全局搜索 `"you will usually use this for PRs"` 在 ZCode 程序代码中命中：
> `~/.zcode/cli/plugins/cache/zcode-plugins-official/browser-use/0.1.0/dist/mcp/server.js:32252`

这句话是一个**硬编码的字符串常量**：

```javascript
var MAIN_BRANCH_LABEL = "Main branch (you will usually use this for PRs)";
```

注入逻辑（`buildGitSystemContextContent` 函数）：

```javascript
function buildGitSystemContextSection(envInfo) {
  if (!isEnvInfoGitRepository(envInfo)) {
    return null;  // 不是 git 仓库 → 不注入（这就是 ZCodeProject 空目录能用的原因）
  }
  const content = buildGitSystemContextContent(envInfo);
  return {
    name: "System Context",
    source: "system_context",
    injectionTarget: "system",   // 注入到 system 角色
    cacheHint: "dynamic",
    // ...
  };
}

function buildGitSystemContextContent(info) {
  const lines = [GIT_SYSTEM_CONTEXT_PREFIX];
  if (info.gitBranch)   lines.push("", `Current branch: ${info.gitBranch}`);
  if (info.gitMainBranch) lines.push("", `${MAIN_BRANCH_LABEL}: ${info.gitMainBranch}`);  // ← 触发拦截
  if (info.gitUser)     lines.push("", `Git user: ${info.gitUser}`);
  lines.push("", `Status:\n${formatGitStatus(info)}`);
  lines.push("", `Recent commits:\n${formatRecentCommits(info)}`);
  return lines.join("\n");
}
```

关键特征：

- **运行时拼接，不写文件**：这段文本在每次发请求前于内存中实时拼合，拼完直接进 HTTP 请求体，不落盘到 AGENTS.md 或任何文件。
- **检测 git 才注入**：`isEnvInfoGitRepository()` 判断是否 git 仓库，是则注入，否则跳过。
- **不可配置**：这个模板文案是编译进 `server.js` 的常量，不在 `config.json`、`setting.json`、`AGENTS.md` 任何用户可编辑的文件里。
- **Hook 也改不了**：ZCode 的 hook 只能通过 `additionalContext` 追加内容，不能改写已注入的 system prompt。

## 四、为什么只有 ZCodeProject 不被拦

| 项目 | 是否 git 仓库 | 注入 gitStatus | 结果 |
|---|---|---|---|
| `ZCodeProject`（空目录） | 否 | 不注入 | ✅ 正常 |
| 其余 13 个项目 | 是 | 注入（含触发句） | ❌ 被拦 |

## 五、解决方案

| 方案 | 做法 | 效果 |
|---|---|---|
| **⭐ 换 provider** | 在 ZCode 模型选择器切到智谱官方（`open.bigmodel.cn`）或 VibeBabo（`vibebabo.com`） | 根治，这两家不拦这句 |
| 给腾讯/ZCode 提反馈 | WorkBuddy WAF 对正常 coding 上下文误报；ZCode 模板措辞触发第三方审查 | 治本但需等修复 |
| ❌ 改 server.js | 直接编辑 `MAIN_BRANCH_LABEL` 常量值 | 不推荐，更新会覆盖 |

## 六、排查方法论

> [!tip] 这次的排查路径
> 1. **读会话上下文**（`ReadSessionContext`）→ 确认输入内容
> 2. **查 ZCode 日志**（`~/.zcode/cli/log/*.jsonl`）→ 确认 HTTP 状态、model、finishReason
> 3. **查 SQLite 数据库**（`~/.zcode/cli/db/db.sqlite`）→ 批量统计 finishReason + tokens 模式
> 4. **读 model-io rollout**（`~/.zcode/cli/rollout/model-io-sess_*.jsonl`）→ 拿到实际发给模型的完整请求体
> 5. **curl 重放 + 二分法** → 逐条 message、逐段、逐句拆解定位触发点
> 6. **全局搜索源码** → 反查触发句在程序中的定义位置和注入逻辑

核心思路：**当用户输入无害但被拦时，问题一定在工具拼接的隐藏 prompt 里。拿到实际请求体（而非用户看到的输入），二分重放是定位根因的最快方法。**
