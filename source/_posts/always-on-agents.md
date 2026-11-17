---
title: 构建始终在线的 Agent
date: 2026-11-17 23:39:12
cover: /images/always-on-agents/fig-01-cycle-poster.webp
tags:
- Agent
- Anthropic
- 翻译
- always-on
categories: 技术
---
本文译自 Anthropic / Claude 的[《Building always-on agents》](https://claude.com/resources/articles/building-always-on-agents)。原文由 William Hu（Applied AI @ Anthropic）撰写，2026 年 11 月 13 日发布。

始终在线的 agent（always-on agent）会无限期运行，并在许多次 session 中自行决定如何行动、何时行动。本指南将介绍它们的核心原语（primitive），以及这些原语如何组合在一起。

<!-- more -->

在过去几个月里，我们看到始终在线的 agent 再度兴起。

始终在线的 agent 是有状态的、主动的：它们跨 session 管理上下文，并在没有直接监督的情况下采取行动。底层来看，它们运行在与其他 agent 相同的[核心循环](https://www.anthropic.com/research/building-effective-agents)上。差别在于外围架构：它由一组原语构成，用来控制 agent 何时醒来、两次 run 之间保留什么，以及它们的工作如何到达用户。

这轮兴起很大程度上来自近期的模型进步，使始终在线的 agent 更切实可跑。首先，最新模型有更好的判断力：它们在理解用户意图、以及知道何时该来核对方面更可靠。其次，智能正变得更便宜，而长周期任务从中获益最多。例如，在一项[长周期编程基准](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)上，Opus 5.5 每个已解决任务的成本比 Opus 5 低 78%，而在典型工作负载上则低 40%。

客户已经开始向我们询问，如何把这类体验集成进他们自己的产品。有些公司希望让现有的 agent 更主动，另一些则打算从零开始构建。

这篇文章会拆解始终在线的 agent 背后的原语，并说明它们如何组合在一起。

## 始终在线的 agent 背后的循环

始终在线的 agent 实际上并不是「始终开着」。这只是对用户体验的一种描述。

现实中，agent 大部分时间都在休眠，只是周期性地醒来做事。每一段工作称为一次 **run**：agent 收到一个事件，完成任务，然后保存所发生事情的信息。一次 run 在 **session** 中进行，session 是消息和工具调用的记录。run 结束后，agent 再次进入休眠，这个循环无限重复。

![始终在线的 agent 时间线动画：大部分时间处于不消耗 token 的休眠，中间穿插短暂的 run。每次 run 有四个步骤：事件到达、session 开始、agent 循环运行，以及 run 写回在多次 run 之间延续的存储状态。](/images/always-on-agents/fig-01-cycle.gif)

## 核心原语

始终在线的 agent 的外围架构有五个核心原语：事件收件箱（event inbox）、定时器（timers）、记忆（memory）、权限（permissions），以及发件箱（outbox）。我们逐一来看。

### 事件收件箱

大多数 AI agent 只在用户发来消息时运行。相比之下，始终在线的 agent 可以被许多类型的事件触发，包括 webhook、定时任务和后台作业。

一种诱人的做法是让每个事件源直接调用 agent，但这有几种失败模式。例如，两个事件同时到达可能启动相互竞争的 run，而一次崩溃则可能彻底丢失一个事件。

这就是事件收件箱的用武之地。每个来源不再直接调用 agent，而是把事件送到收件箱，由 worker 来取走。

最简单的版本有三个组成部分：

- 一份耐久的事件列表，通常是一张数据库表；
- 一个处理器（handler），通常是 HTTP 端点，把事件写入该列表；以及
- 一个 worker，通常是后台服务，从该列表中读取。

收件箱应跟踪每个事件的状态，并且可以依据来源提供的唯一标识符对事件去重。关系型数据库很适合做收件箱，每个事件作为新行追加。我们建议先按每个用户一个收件箱起步，如果需要更高并行度，可以再进一步拆分。

![事件收件箱示意图。处理器（HTTP 端点）把每个事件写入收件箱（一张跟踪状态并丢弃重复项的数据库表），并唤醒 worker。Worker 读取待处理项，启动 run，等待其完成，再把事件标记为完成。](/images/always-on-agents/fig-02-event-inbox.webp)

当新事件到来时，处理器把事件写入收件箱并唤醒 worker。Worker 读取所有待处理项并启动一次 run。那些延续先前工作的事件（例如用户的回复）会恢复已有 session，其余事件则开启新的 session。

此后，worker 把事件交给 agent 循环，等待 run 结束，再把相应事件标记为完成。实现细节可能比较棘手，下面是几条最佳实践：

- **处理器应尽快确认事件。** 有些来源（例如 webhook 提供方）会把过慢的响应视为失败。处理器仍可以做轻量检查，但任何更深入的校验（例如检查提示注入）都应在 run 期间进行。
- **收件箱应作为事实来源。** Worker 被唤醒时，应先从收件箱拉取最新事件。这样，如果 worker 在 run 中途挂掉，新的 worker 可以取走同一个事件并重试。
- **一个收件箱同一时间只应有一个活跃的 worker。** 这样可以防止两次 run 重复劳动或互相覆盖。当 run 进行到一半又出现新事件时，你可以把它推到下一次 run、并入当前 run，或者中断并重启。
- **Worker 应将待处理事件打包在一起。** 例如，如果用户连续发了几条消息，worker 可以把它们合并成一次 run，这样 agent 只需要回复一次。

### 定时器

与事件收件箱一起，定时器让始终在线的 agent 能够主动行动。定时器通常分为两类：跟进（follow-up）或日程（schedule）。跟进是 agent 给自己设的截止日期，例如去查看一封回复。日程则是按已知节奏重复发生的任务，例如每日通讯。

定时器存放在自己的耐久存储中，通常是另一张数据库表，与收件箱分开。收件箱应只保存已经可以处理的事件。跟进和日程可以共用相似的模式（schema），但日程还需要跟踪其重复规则。

![定时器示意图。一次 run 在数据库表中设置定时器，既可以是 agent 设置的跟进，也可以是用户设置的日程。调度器找出到期项，并向事件收件箱添加事件，从而启动下一次 run。](/images/always-on-agents/fig-03-timers.webp)

定时器本身并不会启动一次 run。相反，调度器（通常是一个后台服务）按固定间隔轮询该存储，并触发所有到期项。每一轮，调度器会：

1. 扫描存储并找出到期的定时器。
2. 根据类别更新每个定时器。
   1. 跟进应被标记为完成。
   2. 日程应把到期时间更新为下一次发生。
3. 向收件箱添加一个事件，并唤醒 worker。

对于调度器，你可以自己构建服务，也可以使用耐久执行引擎。

### 记忆

始终在线的 agent 要跨越许多彼此断开的 run 工作，并依靠记忆来保持连续性。

与事件收件箱和定时器类似，记忆也应有自己的存储。在我们的产品中，例如 [Claude Code 的自动记忆](https://code.claude.com/docs/en/memory) 和 Messages API 的[记忆工具](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)，这是以一个文件目录来实现的。对始终在线的 agent 来说，这些目录应映射到耐久存储，通常是数据库或对象存储。这样，记忆就能比 worker 活得更久。

作为经验法则，记忆应只保存 agent 无法再查到的内容。如果某样东西已经存在于另一个系统（例如问题跟踪器），你应保存一个指针。下面是始终在线的 agent 应存储的记忆类型：

- **用户指令：** 用户关于 agent 行为的任何指令，都应保存在单独的文件中，并在每次 run 开始时加载。用户并不总会在场来引导 agent，因此这些指令必须在多次 run 之间延续。
- **用户笔记：** Agent 应保存它关于用户学到的内容，例如他们的角色、偏好，或他们给出的任何纠正。你可以先从单个文件开始，随着笔记增多再拆成若干小文件。在后一种情况下，每次 run 开始时只加载一份索引，但 agent 需要更多细节时可以把单个文件拉进上下文。
- **过往消息：** Agent 应跟踪自己已经告诉过用户的内容，以免重复。它可以把这些存在每次 run 开始时加载的单独文件里，也可以由 worker 加载 agent 与用户的最近几条消息。
- **任务进度：** 对每项进行中的任务，agent 应保留一份进度文件，说明已完成什么、还剩什么。之后的新 run 就可以读这份文件，并从上次停下的地方继续。与用户笔记类似，每次 run 开始时只加载一份索引，但 agent 需要更多细节时可以把单个文件拉进上下文。

下图展示了一种示例布置。

![把记忆组织成文件目录的示例。每次 run 开始时加载 instructions.md、index.md 和 reported.md，而 notes/ 与 progress/ 文件夹在需要时再读取。Agent 在 run 期间写入记忆。](/images/always-on-agents/fig-04-memory.webp)

以上描述的都是在一次 run 期间保存的记忆。你也可以加入 dreaming（梦境整理）：回顾过往 session 来策展记忆。Dreaming 可以替换过时条目，并发现只有跨越许多次 run 才会出现的模式。若想了解更多，我们关于 [agent 记忆](https://www.anthropic.com/research/components-of-agent-memory) 的文章有更详细的讨论。

#### 在上下文窗口之内

在开始做任何工作之前，agent 会加载系统提示、工具定义和记忆文件。这种初始的上下文「开销」会出现在每一次 run 中，因此我们建议保持精简，并使用[提示缓存](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)来降低成本。

新的 session 一开始只有这些上下文开销，再加上触发本次 run 的事件。而已有的 session 则会带回先前 run 中的消息和工具调用。

如果一个 session 变得太长，你可以[压缩较早的消息](https://platform.claude.com/docs/en/build-with-claude/compaction)，或开启一个新的 session。两种做法都会损失细节，因此需要持久保留的内容，应沿途写入记忆。

![随时间变化的上下文用量动画图。系统提示、工具和记忆构成的固定带宽出现在每次 run 中。已有 session 会带回完整记录并持续增长，直到压缩把较早的记录变成摘要。](/images/always-on-agents/fig-05-context.gif)

### 权限

权限是始终在线的 agent 风险最高的地方。一次 run 持有用户的访问权限，并且从设计上就是无人值守运行的。对每次工具调用，agent 需要决定是自行继续、拦截，还是等待批准。

权限所用的组件，与其他 agent 使用的是同一套：

- **凭证存储：** 始终在线的 agent 往往需要敏感的用户凭证（如密码或访问 token）才能发挥作用。[推荐的模式](https://code.claude.com/docs/en/agent-sdk/secure-deployment#credential-management)是在 agent 环境之外放置一个代理，由它为出站请求添加凭证。Agent 发出的请求不含凭证，代理在转发前注入真实凭证。
- **自动模式（auto mode）：** 始终在线的 agent 应能在没有用户持续输入的情况下工作。我们已经在 Claude Code 的[自动模式](https://www.anthropic.com/research/claude-code-auto-mode)中处理过这个问题的一个版本。自动模式使用分类器审查较高风险的操作，并拦截它们认定为危险的操作，以便 Claude 寻找更安全的路径。不过，对于最重要的决定，用户审查仍然是必要的。
- **延迟的工具调用（deferred tool calls）：** 在 Claude Code 中，hook 可以[推迟一次工具调用](https://code.claude.com/docs/en/hooks#defer-a-tool-call-for-later)，等待用户提供额外输入。同样的模式也适用于始终在线的 agent：它们在采取某些行动之前，可能需要澄清或批准。当一次调用被推迟时，run 结束，session 在调用中途被保存。请求会展示给用户（例如通过用 [MCP Apps](https://modelcontextprotocol.io/extensions/apps/overview) 构建的交互卡片），用户的答复则作为事件回到收件箱。该事件启动一次新的 run，用用户的输入恢复已保存的 session。在延迟调用等待响应期间，worker 可以拾取其他事件，因此 agent 不会被堵住。

### 发件箱

对于大多数 AI agent，响应会在生成过程中流式传输给用户，因此他们可以跟着 Claude 一起推理问题、调用工具并写出答案。

始终在线的 agent 则大部分工作在后台完成。用户得到的是结果和少量细节，而不是完整的逐步拆解。

![发件箱示意图。一次 run 可以以无话可说、一次发送工具调用，或一次向用户提问的延迟工具调用结束。消息进入发件箱（一张数据库表），由分发器交付给用户。延迟请求既可以通过发件箱，也可以显示在你自己的界面中。](/images/always-on-agents/fig-06-outbox.webp)

一次 run 可能以三种方式结束：

- Agent 向发件箱发送一条消息，
- Agent 判断没有什么需要发送，或
- Agent 使用延迟的工具调用，请求权限或额外输入。

大多数始终在线的 agent 用一个「发送」工具，把消息直接交给用户，就可以应付。只有当你希望调节 agent 的消息节奏，或让 agent 能从错误中恢复时，才需要一个正式的发件箱。

发件箱是一张存放待发消息的独立表，与事件收件箱类似。另外还有一个分发器（dispatcher），通常是后台服务，类似于 worker。分发器读取待处理消息，决定交付哪些，并把它们发送给用户。来自延迟工具调用的请求，既可以通过发件箱，也可以直接显示在界面中。

除了交付之外，调节 agent 发送多少消息也很重要。发得太多会显得吵闹，发得不够则会漏掉用户需要知道的事情。我们建议通过构建评估（eval）来找到合适的平衡。实践中，最好让 agent 先从安静起步，再随着时间逐渐赢得更多用户注意力。

## 它们如何组合在一起

五个原语是这样组合在一起的：

![五个原语如何组合的动画示意图。定时器和事件收件箱启动一次 run，该 run 在 session 中使用记忆和权限，发件箱把消息交付给用户，用户的答复再作为事件回来。](/images/always-on-agents/fig-07-primitives.gif)

每一次 run 都从收件箱中的一个事件开始，无论它是用户消息、webhook，还是定时器。当 worker 取到一个事件时，它会启动一次 run，并在新的或已有的 session 中唤醒 agent。Agent 读取记忆以获得基线，在其权限范围内采取行动，再把学到的内容更新回记忆。run 结束后，agent 向发件箱推入一行，分发器再把消息交付给用户。

它们一起构成一个循环。一次 run 可以设置稍后再次唤醒 agent 的定时器；当用户回复一条消息时，他们的答复会作为事件回到收件箱。每一次 run 都很短暂，但这个循环会无限持续下去，而这正是让 agent 感觉始终在线的原因。

## 致谢

*由 William Hu 撰写。特别感谢 Rodrigo Olivares 和 Michael Segner 的贡献。*
