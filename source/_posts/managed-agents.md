---
title: 规模化 Managed Agents：将大脑与双手解耦
date: 2026-11-17 23:12:45
cover: /images/managed-agents/fig-01-interfaces.webp
tags:
- Agent
- Anthropic
- 翻译
- harness
categories: 技术
---
本文译自 Anthropic 的[《Scaling Managed Agents: Decoupling the brain from the hands》](https://www.anthropic.com/research/managed-agents)。原文由 Lance Martin、Gabe Cemaj 与 Michael Cohen 撰写，2026 年 4 月 8 日发布于 Anthropic Engineering。

*要开始使用 Claude Managed Agents，请参阅他们的[文档](https://platform.claude.com/docs/en/managed-agents/overview)。*

工程博客（Engineering Blog）上一个持续讨论的主题，是如何[构建有效的 agent](https://www.anthropic.com/engineering/building-effective-agents)，以及如何为[长时运行的工作](https://www.anthropic.com/engineering/harness-design-long-running-apps)[设计 harness](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)（Harness，即围绕模型的编排运行时）。这些工作中有一条共同线索：harness 会把「Claude 自己做不到什么」编码成各种假设。然而这些假设必须经常被重新审视，因为随着模型进步，它们会[过时](http://www.incompleteideas.net/IncIdeas/BitterLesson.html)。

<!-- more -->

仅举一例：在先前的工作中我们[发现](https://www.anthropic.com/engineering/harness-design-long-running-apps)，Claude Sonnet 4.5 会在察觉自己接近上下文上限时过早收尾——这种行为有时被称为「上下文焦虑」（context anxiety）。我们的应对是在 harness 中加入上下文重置。但当我们把同一套 harness 用在 Claude Opus 4.5 上时，这种行为已经消失了。重置反而成了多余负担。

我们预计 harness 还会继续演进。因此我们构建了 Managed Agents：这是 Claude Platform 中的一项托管服务，通过一组意在比任何具体实现（包括我们今天运行的实现）更长久的接口，代你运行长周期 agent。

构建 Managed Agents，意味着要解决计算领域的一个老问题：如何为一个「[尚未想到的程序](http://www.catb.org/esr/writings/taoup/html/ch03s01.html)」（programs as yet unthought of）设计系统。几十年前，操作系统把硬件虚拟化成——*进程、文件*——这类足够通用的抽象，以容纳当时还不存在的程序，从而解决了这个问题。抽象比硬件更长久。`read()` 命令并不关心自己访问的是 1970 年代的磁盘组还是今天的 SSD。上层抽象保持稳定，底层实现可以自由更换。

Managed Agents 遵循同样的模式。我们把 agent 的组成部分虚拟化了：session（会话，记录所发生一切的只追加日志）、harness（调用 Claude、并把 Claude 的工具调用路由到相应基础设施的循环），以及 sandbox（沙箱，供 Claude 运行代码和编辑文件的执行环境）。这样，每一部分的实现都可以替换，而不扰动其他部分。我们在这些接口的形态上是有立场的，但不规定接口背后跑的是什么。

![Managed Agents 接口示意图：Session、Harness、Sandbox、Tools 与 Orchestration](/images/managed-agents/fig-01-interfaces.webp)

## 不要收养一只「宠物」

一开始，我们把 agent 的所有组件放进同一个容器，也就是说 session、agent harness 和 sandbox 共享一个环境。这种做法有好处，包括文件编辑是直接的系统调用，而且不必设计服务边界。

但把一切耦合进一个容器后，我们碰上了基础设施里的老问题：我们收养了一只[宠物](https://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/)（pet）。在「宠物 vs 牲畜」（pets-vs-cattle）的类比里，宠物是有名字、需要亲手照料、丢不起的个体，而牲畜是可互换的。在我们这里，服务器成了那只宠物；容器一旦失败，session 就丢了。容器一旦无响应，我们就得把它「护理」回来。

护理容器意味着要调试无响应、卡住的 session。我们唯一的观察窗口是 WebSocket 事件流，但它无法告诉我们故障出在哪里，于是 harness 里的 bug、事件流中的丢包，或容器下线，表现都一样。要弄清出了什么问题，工程师得打开容器里的 shell；可那个容器往往还装着用户数据，这种做法实际上意味着我们几乎没有调试能力。

第二个问题是，harness 假定 Claude 所处理的一切都和它住在同一个容器里。当客户要求我们把 Claude 连到他们的虚拟私有云（VPC）时，他们要么把自己的网络与我们的对等互联（peer），要么在自己的环境里运行我们的 harness。写进 harness 的假设，在我们想把它接到不同基础设施时变成了障碍。

## 将大脑与双手解耦

我们最终采用的方案，是把我们所谓的「大脑」（Claude 及其 harness）与「双手」（执行动作的沙箱和工具）以及「session」（会话事件日志）解耦。每一项都成为对其他部分几乎不做假设的接口，每一项都可以独立失败或被替换。

**Harness 离开容器。** 把大脑与双手解耦，意味着 harness 不再住在容器里。它调用容器的方式，与调用任何其他工具一样：`execute(name, input) → string`。容器变成了牲畜。如果容器死了，harness 把失败当作工具调用错误捕获，并回传给 Claude。如果 Claude 决定重试，可以用标准配方重新初始化一个新容器：`provision({resources})`。我们再也不必把失败的容器护理回健康状态。

**从 harness 故障中恢复。** Harness 也变成了牲畜。因为 session 日志位于 harness 之外，harness 里没有任何东西需要在崩溃后存活。当一个 harness 失败时，可以用 `wake(sessionId)` 启动一个新的，再用 `getSession(id)` 取回事件日志，并从最后一条事件继续。在 agent 循环期间，harness 通过 `emitEvent(id, event)` 写入 session，以保留一份耐久的事件记录。

![各组件接口及其可由何种实现满足的对照表](/images/managed-agents/fig-02-interfaces-table.webp)

**安全边界。** 在耦合设计里，Claude 生成的任何不受信任的代码，都和凭证跑在同一个容器里——因此一次提示注入只需要说服 Claude 去读自己的环境。攻击者一旦拿到这些 token，就可以拉起新的、不受限制的 session，并把工作委派给它们。收窄权限范围是显而易见的缓解手段，但这等于把「Claude 拿着受限 token 做不到什么」编码成假设——而 Claude 正变得越来越聪明。结构性的修法，是确保这些 token 永远无法从 Claude 生成代码所运行的沙箱里被触及。

我们用两种模式来保证这一点。认证（auth）可以绑在资源上，也可以放在沙箱外的保险库（vault）里。对于 Git，我们在沙箱初始化时用每个仓库的访问 token 克隆仓库，并把它接到本地 git remote。在沙箱内部就可以使用 git `push` 和 `pull`，而 agent 本身从不经手 token。对于自定义工具，我们支持 MCP，并把 OAuth token 存在安全保险库中。Claude 通过专用代理调用 MCP 工具；该代理接收与 session 关联的 token，再从保险库取出对应凭证，向外服务发出调用。Harness 始终不会获知任何凭证。

## Session 并不是 Claude 的上下文窗口

长周期任务常常超出 Claude 上下文窗口的长度，而常见的应对方式都涉及对「该保留什么」作出不可逆的决定。我们在先前关于上下文工程（context engineering）的[工作](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)中探索过这些技术。例如，压缩（compaction）让 Claude 保存其上下文窗口的摘要，记忆工具（memory tool）让 Claude 把上下文写入文件，从而可以跨 session 学习。这可以与上下文裁剪（context trimming）搭配，有选择地移除旧的工具结果或思考块等 token。

但不可逆地选择保留或丢弃上下文，可能导致失败。很难知道未来的轮次会需要哪些 token。如果消息在压缩步骤中被变换，harness 会把已压缩的消息从 Claude 的上下文窗口里移除，只有在它们被存储的情况下才能恢复。先前的研究已经[探索](https://arxiv.org/pdf/2512.24601)过应对办法：把上下文存成一个活在上下文窗口*之外*的对象。例如，上下文可以是 REPL 中的一个对象，LLM 通过写代码来过滤或切片，以编程方式访问它。

![Session 与 Harness 之间通过 Events 和 getEvents 交互](/images/managed-agents/fig-03-session-harness.webp)

在 Managed Agents 中，session 提供了同样的好处：它作为活在 Claude 上下文窗口之外的上下文对象。但上下文不是存在沙箱或 REPL 里，而是耐久地存在 session 日志中。接口 `getEvents()` 允许大脑通过选取事件流的位置切片来查询上下文。这个接口可以灵活使用：大脑可以从上次读到的位置继续，也可以在某个特定时刻之前回退几条事件以查看前因，或在某次具体动作之前重读上下文。

取回的事件也可以在进入 Claude 的上下文窗口之前，于 harness 中被变换。这些变换可以是 harness 所编码的任何内容，包括为了获得高提示缓存命中率而做的上下文组织，以及上下文工程。我们把「可恢复的上下文存储」放在 session 里，把「任意的上下文管理」放在 harness 里，是因为我们无法预知未来模型需要怎样的具体上下文工程。接口把上下文管理推给 harness，只保证 session 是耐久的、可供查询的。

## 许多大脑，许多双手

**许多大脑。** 把大脑与双手解耦，解决了我们最早的客户抱怨之一。当团队希望 Claude 针对他们自己 VPC 中的资源工作时，唯一的路径是把他们的网络与我们的对等互联，因为装着 harness 的容器假定所有资源都在它旁边。一旦 harness 不再位于容器里，这个假设就消失了。同一项改动还有性能上的回报。当初把大脑放进容器，意味着有多少大脑就需要多少容器。对每一个大脑来说，在容器完成置备之前都无法进行推理；每个 session 都要预先支付完整的容器启动成本。即便是那些永远不会碰到沙箱的 session，也必须克隆仓库、启动进程、从我们的服务器拉取待处理事件。

这段空等时间体现在 time-to-first-token（TTFT，首个 token 耗时）上，它衡量的是一个 session 从接受到工作到产出第一个响应 token 要等多久。TTFT 是用户感受最敏锐的延迟。

把大脑与双手解耦之后，容器只在需要时才由大脑通过工具调用 `(execute(name, input) → string)` 来置备。因此，一个并不立刻需要容器的 session 不必等待容器。只要编排层从 session 日志中拉到待处理事件，推理就可以开始。采用这种架构后，我们的 p50 TTFT 大约下降了 60%，p95 下降超过 90%。扩展到许多大脑，只意味着启动许多无状态的 harness，并仅在需要时把它们接到双手上。

**许多双手。** 我们也希望每个大脑都能连接到许多双手。实际上，这意味着 Claude 必须对许多执行环境进行推理，并决定把工作送到哪里——这比在单一 shell 中操作更困难的认知任务。我们一开始把大脑放在单个容器里，是因为更早的模型还做不到这一点。随着智能提升，单个容器反而成了限制：那个容器一旦失败，大脑伸向的每一只手的状态都会丢失。

把大脑与双手解耦后，每一只手都成为工具 `execute(name, input) → string`：名字和输入进去，字符串返回来。这个接口支持任何自定义工具、任何 MCP 服务器，以及我们自己的工具。Harness 并不知道沙箱是一个容器、一部手机，还是一个《宝可梦》模拟器。而且因为没有任何手与任何一个大脑耦合，大脑可以把双手传递给彼此。

![多个 Harness 与多个 Sandbox 相互连接](/images/managed-agents/fig-04-many-brains-hands.webp)

## 结语

我们面对的挑战是一个老问题：如何为一个「尚未想到的程序」设计系统。操作系统能沿用数十年，靠的是把硬件虚拟化成对当时还不存在的程序也足够通用的抽象。通过 Managed Agents，我们希望设计一套能够容纳未来的 harness、sandbox 或其他围绕 Claude 的组件的系统。

Managed Agents 正是同样精神下的元 harness（meta-harness），并不对 Claude 未来需要怎样的具体 harness 预先表态。相反，它是一套带有通用接口、能够容纳许多不同 harness 的系统。例如，Claude Code 是一套优秀的 harness，我们在各类任务中广泛使用。我们也表明，面向特定任务的 agent harness 在狭窄领域表现出色。Managed Agents 可以容纳其中任何一种，从而随时间跟上 Claude 的智能。

元 harness 设计意味着对 Claude 周围的接口有立场：我们预期 Claude 需要操纵状态（session）和执行计算（sandbox）的能力。我们也预期 Claude 需要扩展到许多大脑和许多双手的能力。我们设计这些接口，是为了能在很长的时间跨度上可靠、安全地运行它们。但我们不对 Claude 将需要多少、或位于何处的大脑或双手做假设。

## 作者与致谢

### 致谢

本文由 Lance Martin、Gabe Cemaj 与 Michael Cohen 撰写。感谢 N.T.A. 和 Jeremy Fox 就这些主题进行的有益讨论。特别感谢 Agents API 团队以及 Jake Eaton 的贡献。
