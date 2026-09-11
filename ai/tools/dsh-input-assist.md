---
tags:
  - AI工具
  - DeepSeek
  - dsh插件
  - 输入补全
  - 错别字检查
  - 工具介绍
aliases:
  - dsh-input-assist
  - dsh 输入助手
  - dsh 补全插件
  - DeepSeek Harness 插件
date: 2026-08-24
status: ready
type: 工具介绍
---

# dsh-input-assist：给 DeepSeek Harness 装上 Copilot 式输入体验

> [!abstract] 一句话
> 一个 dsh Web UI 插件：打字停一停，**灰色补全建议内联出现在光标后**（Tab 逐词采纳）；中文**错别字当场标红**、点击修正。装完即用，词典层检查零成本。
>
> 官网：[dsh-input-assist.honlnk.com](https://dsh-input-assist.honlnk.com/) ｜ GitHub：[honlnk/dsh-input-assist](https://github.com/honlnk/dsh-input-assist)

---

## 一、它解决什么问题

dsh 的输入框很干净——干净到写长提示词时什么帮助都没有：没有补全，没有校对。一段三百字的中文提示词里混进一个「迫不急待」，要等模型回复了才被发现，改完还得整段重发。

这个插件把两件现代输入体验的标配带到 dsh 里，且**不占用输入框内的任何空间**：

![整体效果：灰色补全建议内联在光标后，错别字红波浪线标注](https://honlnk-obsidian.honlnk.com/images/20260911_175625_48ced62597e80947.png)

## 二、两个功能

### 输入补全（ghost text）

停笔约 0.8 秒后，灰色建议**直接内联在光标后**——和 Copilot / Cursor 同款手感，不是弹窗，不是悬浮条：

![ghost 建议内联显示 + 错别字导航条同屏](https://honlnk-obsidian.honlnk.com/images/20260911_175624_4fc6394b6fdd34a1.png)

| 动作 | 快捷键 |
| --- | --- |
| 采纳一词（中文按词切分） | `Tab` |
| 全量采纳 | `Shift+Tab` |
| 关闭建议 | `Esc` |

细节都按中文输入调过：逐词采纳按词而不是按字符；吃光建议会自动续写下一轮；输入法组字期间所有快捷键放行，绝不干扰打字。

### 错别字检查（两层）

![错别字导航条：迫不急待 → 迫不及待，当前项高亮](https://honlnk-obsidian.honlnk.com/images/20260911_175623_c9bfa89c6f824666.png)

| 层 | 检什么 | 速度 | 成本 |
| --- | --- | --- | --- |
| 词典层 | ~150 条高置信错词 + 上下文规则（登陆/登录、戴口罩…） | 即时 | **零**（浏览器本地） |
| LLM 层 | 需要句意的混淆（在/再、的/得/地） | ~0.8s | 按 token 计费 |

检出后在文中标红波浪线，**当前选中项加重高亮**：

| 动作 | 快捷键 |
| --- | --- |
| 下一个 / 上一个错字 | `Ctrl+Shift+.` / `Ctrl+Shift+,` |
| 修正当前选中项 | `Ctrl+Shift+F` |
| 标记该处为正确输入 | `Ctrl+Shift+G` |
| 本次输入不再提醒 | `Esc` |

两个安心设计：**修正只替换当前选中的一条**，绝不全量替换；代码块、行内代码、URL 里的内容永远不标——不会把你代码里的「变量名误写」当错别字。

## 三、三分钟装上

```shell
dsh plugin --profile web add dsh-input-assist   # 或 @honlnk/dsh-input-assist
dsh --profile web                                # 重启，打开 http://127.0.0.1:3080
```

API key 不用配置——插件会自动复用 dsh 已保存的 `DEEPSEEK_API_KEY`。想单独配置也可以：`~/.dsh/settings.yaml` 里写 `input-assist.completionApiKey`，或设环境变量。

输入框旁有「补 / 校」两个开关，随时关掉任一功能：

![补 / 校 功能开关（蓝色圆底白字）](https://honlnk-obsidian.honlnk.com/images/20260911_175625_1acc695ec4c580e0.png)

## 四、要花多少钱

- **词典层校对：完全免费**（纯本地扫描，不发任何请求）
- 补全和 LLM 层走 DeepSeek API（deepseek-chat）：补全每次上限 64 token，校对单次上限 800 token；停笔防抖内连续打字只发一次请求
- 想零开销：settings 里 `proofreadUseLlm: false`，只剩词典层，分文不花

## 五、想看实现的

插件是 dsh 的标准双侧结构（host 半边 + 浏览器半边），补全建议用镜像层渲染、错字检查的词典层跑在浏览器本地。完整的设计文档和 v1–v4 开发记录在仓库 [docs/](https://github.com/honlnk/dsh-input-assist/tree/main/docs) 目录，TypeScript 源码在 `src/`。

## 链接

- 官网：[dsh-input-assist.honlnk.com](https://dsh-input-assist.honlnk.com/)
- GitHub：[honlnk/dsh-input-assist](https://github.com/honlnk/dsh-input-assist)（MIT）
- npm：[dsh-input-assist](https://www.npmjs.com/package/dsh-input-assist) / [@honlnk/dsh-input-assist](https://www.npmjs.com/package/@honlnk/dsh-input-assist)
- 上游：[deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)

---

> 非官方社区插件，与 DeepSeek 官方无隶属关系。
