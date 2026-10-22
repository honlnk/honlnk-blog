# 你早就在"跑图"了：Graph、ReAct 与 Agent 框架祛魅

>**日期**：2026年9月20日
>
>**作者**：Honlnk/鸿影
>
>**适用读者**：被 Agent 框架文档劝退过的开发者、手写 agent 循环但没用过框架的人、对 LangGraph / Spring AI Alibaba 感兴趣的读者
>
>**定位说明**：这不是一篇 Graph 教程——概念清单官方文档写得比我全。本文记录一条"先抄项目、后看文档"的祛魅路径：手写 agent 循环的人其实早就在跑图，框架只是把图显式化。顺带澄清 ReAct 与前端 React 的撞名问题，并给 Spring AI Alibaba 文档里的术语做一张翻译表。

---

## 一、起点：一个看不懂文档的人，抄出了一个 Agent 引擎

我最早学 Agent 开发时，读的文档是 Spring AI Alibaba（SAA）。这个框架里 Graph 单独成章，快速开始的主角叫 ReactAgent，文档里铺满了 StateGraph、OverAllState、saver、threadId、Hook 这类名词。当时看得很混乱，看了很久都没学懂。

后来我换了个方式：不看文档了，照着别的项目抄，手写出了自己项目（NovAI）的 Agent 引擎——query 主循环、事件流、工具执行层、权限确认，全部手写。

神奇的事情发生在抄完之后：回头再看那些文档名词，每一个都对号入座了。原来我一直在做的事情，就是框架里叫 "graph" 的那个东西——**我一直在跑图，只是不知道它叫图。**

这篇文章就是把这次"名词归位"的过程记录下来。

---

## 二、Agent 的本质是循环

抛开所有框架，一个能调工具的 Agent，执行逻辑就是这个循环：

```text
LLM 思考 → 决定调用工具 → 执行工具 → 把结果喂回 LLM → 再思考 → …… → 不再调工具，输出答案
```

手写实现就是一个 while 循环加一个 if 判断："这次模型输出里有没有工具调用？有就执行完把结果塞回上下文，继续循环；没有就退出。"调几个工具、走几轮、什么时候停，全是运行时才由模型决定的。我自己项目的引擎是这个结构，后来分析 ZCode 的仓库 wiki 生成功能时，发现它内部的生成 agent 也是同一个结构。

**这个循环就是 agent 范式的血肉。所有框架的存在，都是因为直接操作这个裸循环在工程上会越来越痛。**

为什么早期的"链（Chain）"抽象不够用？链是一条固定管道：A 接 B 接 C，有向无环，开发者把流程完全写死。而上面的循环要求**绕回去再来一轮**，直线管道表达不了环。于是 2024 年初 LangChain 推出 LangGraph，把执行建模成允许有环的图，之后各框架跟进，"graph" 成了 Agent 框架的通用概念。

---

## 三、Graph：把循环显式化

### 3.1 三要素

Graph 是把 agent 的运行过程显式建模成一张有向图：

- **State（状态）**：一个全局状态对象（比如对话消息列表），贯穿整张图，每个节点读取它、更新它（通过 reducer 合并，比如新消息追加而非覆盖）；
- **Node（节点）**：一个计算单元——一次 LLM 调用、一次工具执行、一段普通函数都行；
- **Edge（边）**：流转规则，分固定边（A 走完必走 B）和**条件边**（运行时由函数看当前状态决定走哪条）。条件边是灵魂——agent 的"自主决策"落到图里，就是 LLM 的输出驱动条件边选路。

最经典的 agent 图就两个节点一条回路：

```text
        ┌──────────────────────────┐
        ▼                          │
  [LLM 节点] ──(输出要调工具)──> [工具节点]
      │
      └──(不再调工具)──> [结束]
```

### 3.2 显式化换来了什么

把循环从代码控制流里拿出来、变成声明式的数据结构，框架就能对这张图做四件事：

1. **可控性**：可能的路径是白名单，不像黑盒 AgentExecutor 那样无法约束；
2. **检查点持久化**：每过一个节点给状态拍快照——断点续跑、崩溃恢复、人工介入（暂停等人审批）、时间旅行调试，全都由此而来；
3. **复杂编排**：分支并行、map-reduce、子图、多 agent（supervisor 分发任务给 worker 节点）都有规范表达；
4. **可视化调试**：图结构可以直接画出来看。

一个冷知识：LangGraph 的执行模型灵感来自 Google 的 Pregel 图计算系统——按"超步"（superstep）推进，一轮内所有活跃节点并行跑完再进下一轮，所以它天生支持节点并行。

### 3.3 隐式的图 vs 显式的图

这是我这趟学习最重要的一个认知：**手写循环和框架跑的循环，执行语义是同一个。** 区别在于——

手写循环是**隐式的图**：节点和边的逻辑写死在代码控制流里，结构从外面看不见；框架是**显式的图**：节点注册进去、边配置出来，图本身成为数据，可以遍历、画出来、拍快照、在任意边上暂停。

所以图不是另一种执行方式，是同一执行方式的结构化表达。**很多人（包括曾经的我）一直在"跑图"，只是没有"建图"。**

### 3.4 一点冷静的反思

2025 年之后社区有个讨论：graph 是不是过度工程？Anthropic 在《Building effective agents》里主张从最简单的形态开始——很多场景一个 while 循环加几个工具就够了。我的看法是：**agent loop 是本质，graph 是它的工程化外壳**。什么时候需要外壳，取决于你要不要复杂编排、状态持久化、人工介入这三件事。理解循环优先于理解图。

---

## 四、ReAct：另一个层面的事

### 4.1 先澄清撞名

SAA 文档快速开始的主角叫 `ReactAgent`——这里的 "React" 和前端的 React 框架**没有任何关系**。它来自 2022 年 Google Brain 的论文《ReAct: Synergizing Reasoning and Acting in Language Models》，是 **Reasoning（推理）+ Acting（行动）** 的合成词。讽刺的是前端 React（2013 年）比这篇论文更早占用这个名字，纯属撞车。

### 4.2 ReAct 的原始形态与现代形态

论文里的 ReAct 是**纯提示词工程**：让模型交替输出 `Thought:（思考）→ Action:（调工具）→ Observation:（观察结果）` 的文本轨迹，用显式的文字推理驱动工具循环。

后来模型有了原生 tool calling 能力，"Thought:"这种文字格式淡化了，但循环结构完整保留。所以今天框架里的 ReactAgent，实际意思是"**实现了那个经典工具调用循环的预制 Agent 类**"——名字来自论文，血肉是 tool calling 时代的。LangGraph 里的对应物是 `create_react_agent` 预构建函数。

---

## 五、ReAct 和 Graph：不是一回事，是两个层面

这是最容易糊的一层窗户纸。两者常被并列提及，看上去都像在说"推理—执行循环"，但判断两个概念是不是一回事，最好的办法是看它们能否**脱离对方独立存在**——这两个都能：

- **ReAct 不需要 graph**：最早的 ReAct 实现就是纯 prompt 加一个 while 循环，没有任何节点和边；我自己的项目也是证据——实现了 ReAct 模式，代码里却没有一张显式的图。ReAct 的本质是**决策节奏**：让模型在"想"和"做"之间交替，用行动结果修正下一步推理；
- **Graph 不需要 ReAct**：图是通用流程引擎，一条直线走到底的 RAG 管道（检索→重排→生成）、带人工审批节点的发布工作流、supervisor 多 agent 编排——都不是"推理—执行循环"，但都可以、也确实在用图建模。图不 care 节点里跑什么逻辑，它只规定流程怎么流转、状态怎么传递、在哪里可以暂停恢复。

两者的交集就是那个类名：**ReactAgent = 用 graph 引擎预实现的 ReAct 循环**。命名上各占一半，恰好暴露了血统。

```text
ReAct（行为范式）                    Graph（实现结构）
"想→做→看→再想"的决策节奏            节点+边+状态的流程引擎
        │                                │
        │   纯 prompt 也能跑              │   线性管道、审批流、
        │   while 循环也能跑              │   多 agent 编排都能建
        │                                │
        └─────── 交集：ReactAgent ───────┘
              用图引擎预装的 ReAct 循环
```

打个比方：ReAct 之于 Graph，就像 MVC 之于 Web 框架——MVC 是架构思想，手写代码也能遵守；Django、Rails 把它显式化成了结构，但框架本身也能写非 MVC 的东西。**思想不依赖载体，载体不专属于思想。**

一句话记忆：**ReAct 说的是"循环"，graph 说的是"把循环画出来"。**

---

## 六、实战翻译表：Spring AI Alibaba 文档词汇归位

有了上面的心智模型，再看 SAA 的文档，当年那些劝退名词全部可以翻译：

先看一个佐证血统的细节：`ReactAgent` 的包路径是 `com.alibaba.cloud.ai.graph.agent`——它就**长在 graph 包上**。这就是 "Graph Core 单独成章"的原因：Graph Core 是通用引擎，ReactAgent 是用这个引擎预装好的一种特定拓扑（两节点一回路）。

| 文档词汇 | 真实身份 |
|---|---|
| `saver(new MemorySaver())` | 检查点（checkpointer），内存版；生产换持久化实现 |
| `threadId` | 会话线程，检查点按它归档，同 threadId 即可续上记忆 |
| `OverAllState` | 图里流动的那个全局状态对象 |
| `RunnableConfig` | 一次运行的配置（携带 threadId、元数据） |
| `ModelCallLimitHook` | 给回路装的最大迭代保险丝，防无限循环 |
| `HumanInTheLoopHook` | interrupt 人工介入（指定工具执行前暂停等审批） |

没有一个是新概念，全是图的能力换了 Java 名字。

另外还有个有趣的发现，可以安慰当年被劝退的自己：SAA 这个快速开始，其实是 **LangChain Python 官方快速开始的 Java 翻译版**——天气工具、双关语天气预报、连 `punnyResponse` 字段都原样照搬。它默认读者已经有 LangGraph 的心智模型；没有的话，混乱是正常的，不全是你的问题。

这也引出我的方法论总结：**官方文档是参考结构**——按概念组织（Graph、State、Hook、Checkpoint 各一章），适合已经懂的人查；而入门需要的是**任务结构**——先照着项目做出一个能跑的东西，让循环的血肉先长出来，再回头把名词逐个归位。先实践后文档，对这个领域可能是更优路径。

---

## 七、结语：把 buzzword 翻译成自己的经验

这个系列的博客做到这里，三篇其实共享同一个主题：wiki 篇回答"这个热点是什么"，skill 篇（[[skills-progressive-disclosure]]）回答"这个机制怎么实现的"，本篇回答"这个概念我早就会了，只是不知道名字"——**都是把 buzzword 翻译成自己已有的经验。**

Agent 领域的名词密度还在膨胀（graph、ReAct、checkpoint、interrupt、supervisor……），但底层的东西变化得很慢：一个循环、一层按需加载的提示词、一份预编译的知识。下次再遇到看不懂的新名词，可以先问一句：它是不是我早就做过的事情，换了个名字？

---

## 参考与延伸

- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)（Yao et al., 2022）——ReAct 原始论文
- [LangGraph 文档](https://langchain-ai.github.io/langgraph/)——graph 抽象的发源地
- [Building effective agents](https://www.anthropic.com/research/building-effective-agents)——Anthropic 对"workflow vs agent"与过度工程的反思
- [Spring AI Alibaba 快速开始](https://java2ai.com/docs/quick-start)——本文第六节的分析对象
- 本系列前两篇：[[llm-wiki-and-deepwiki|AI 圈的 "Wiki" 到底是什么]]、[[skills-progressive-disclosure|Skill 的实现原理：渐进式披露]]
