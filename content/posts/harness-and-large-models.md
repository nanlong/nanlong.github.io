---
title: "Harness 会被大模型吃掉吗？"
date: 2026-07-28T10:00:00+08:00
draft: false
tags: ["AI", "大模型", "Agent", "Harness", "Claude Code", "Codex"]
categories: ["技术"]
description: "把模型和 Harness 放进三组对照里，看看哪些代码会消失，哪些只是换了位置。"
---

先看三个有点矛盾的现象。

同一个 Claude 模型，通过 API 调用和放在 Claude Code 里用，完成编程任务的能力可以差很多。

模型一代比一代擅长规划和使用工具，Claude Code 与 Codex 包在模型外面的东西却没有跟着消失。它们开始管理上下文、执行命令、隔离权限、保存会话，甚至组织多个 Agent 一起工作。

到了具体项目里，我们还在写 `CLAUDE.md` 和 `AGENTS.md`，还要准备测试、日志、浏览器、沙箱和审批规则。模型越强，周围要准备的东西似乎越多。

这就很奇怪了。

如果 Harness 只是模型能力不足时用来撑场面的脚手架，模型变强之后，它应该逐渐变薄才对。可我们看到的，一半符合这个推断，另一半正好相反。

问题到底出在哪里？

<!--more-->

## 先提出一个可以被推翻的猜想

中文互联网上流传的「Harness 是死路」，有一个可以核验的出处。

2026 年 5 月 5 日，「马克的技术工作坊」发布了视频《Harness Engineering 到底是什么？概念、实战与争议，一次全部讲清楚》。在 27:22 开始的争议部分，他把反对者的看法概括为：

> 所有的 Harness Engineering 最终都会被淘汰。

视频作者自己的结论稍微保守一些：Harness Engineering「不是噱头，但大概率也不是终局」。〔1〕

所以，「Harness 是死路」是这场中文讨论里的压缩说法，不能挂到 Anthropic、OpenAI 或某位公司负责人的头上。

在英文世界，Nicolas Bustamante 的表达更直接。他在《Hard Lessons Building Agents Since GPT-3.5》的「The Bitter Lesson of Scaffolding」一节写道：

> Scaffolding will not survive AGI.

他的理由也很好懂：开发者为了弥补当代模型的缺陷，不断在外围增加路由、提示词、状态机和纠错流程。模型升级以后，这些补丁可能突然失去作用，甚至妨碍新模型。〔2〕

我们可以把这个观点改写成一个方便检查的猜想：

> 如果一段 Harness 代码只是在补偿模型的能力，那么只换一个更强的模型，这段代码的价值就会下降。

这个猜想不需要靠立场来证明。找几组对照看看就行。

## 第一组对照：只换模型

先保持任务不变，只把模型换掉。

早期大模型很难稳定输出 JSON。应用拿到结果后，要截取代码块、修补引号、删除多余逗号，失败了还得换一种提示词重试。结构化输出进入模型 API 以后，这批修复代码很快变成了负担。

工具选择也走过相似的路。模型分不清几十个工具时，开发者会先写一层关键词路由：看到「天气」就去天气服务，看到「邮件」就去邮箱。后来模型能直接理解工具描述，这层路由不但没有帮助，还可能把一个本来正确的请求送错地方。

固定的 Planner、Executor、Reviewer 工作流也值得怀疑。它们常常把一次任务硬拆成几个角色，再用大量格式约束让角色互相传话。模型的长程规划和自我检查能力提高以后，有些多 Agent 流程还不如让一个模型连续做完。

Fable 5 是一个更极端的例子。以前让 Claude 玩《宝可梦 火红》，需要地图、导航辅助和额外的游戏状态。Fable 5 依靠原始画面和更薄的视觉 Harness 完成了游戏。原来放在模型外面的环境理解，确实有一部分进入了模型。〔6〕

Opus 5 在解析器缺少真实行情验证条件时，会主动搭出测试 Harness 检查自己的实现。GPT‑5.6 Sol 则把程序化工具调用和并发子 Agent 带进 API。过去需要应用编排的动作，模型已经能承担得更多。〔6〕

这一组结果支持了最初的猜想。

正则修复器、关键词路由、僵硬的角色分工，都记录了某一代模型的弱点。弱点消失，代码便失去了存在的理由。它们当然可以叫 Harness，但更准确地说，是模型的认知补丁。

如果所有 Harness 都属于这一类，文章写到这里就可以结束了。

可现实里还有另一组对照。

![随着模型升级，JSON 修复、工具路由和固定工作流等认知补丁被吸收到模型内](/images/posts/harness-and-large-models/cognitive-scaffolding.webp)

## 第二组对照：模型不换，只换 Harness

2026 年 4 月，Claude Code 出过一次很适合观察这个问题的故障。

一处上下文恢复错误丢掉了部分推理历史。Claude 仍然收到对话，却看不到自己先前为何作出某个决定。另一处 system prompt 长度限制又让部分评测下降了 3%。那段时间，一些用户感觉 Claude Code 变得健忘，重复调查已经看过的问题，还会作出奇怪的工具选择。〔3〕

值得留意的是，Claude API 没有受到影响。

模型没有随这次事故变笨。出问题的是模型外面的上下文管理，用户感受到的 Agent 能力却随之下降。

这相当于把第一组对照反过来做了一次：模型尽量保持不变，只改动 Harness。

![同一个模型在完整 Harness 与上下文恢复错误下的运行回路对照](/images/posts/harness-and-large-models/context-loop-comparison.webp)

Claude Code 平时做的事，大致是这样的：先读取项目文件和 `CLAUDE.md`，把相关内容放进上下文；模型决定下一步查看什么、运行什么；Claude Code 真的去执行，再把结果交还给模型。这个循环会持续很多轮。对话如何恢复、命令是否允许、输出保留多少，都发生在模型权重之外。〔4〕

于是，原先的猜想开始不够用了。

模型更会选工具，确实可以让某些路由代码消失。但工具仍要有人执行。模型更能理解长对话，也不能自己保存会话。它可以判断一条命令有风险，却不能靠这个判断夺走自己访问生产环境的权限。

Anthropic 在事故复盘里还提到，他们会针对不同模型调整 Claude Code 的 Harness。旧模型需要的提醒，新模型可能觉得啰嗦；新模型对上下文的使用方式变了，恢复策略也要跟着改。

所以，模型升级之后发生的事情，并不是 Harness 按比例缩小。更常见的情况是：一批补丁被删掉，另外一些职责变得重要。

到这里还不能下结论。也许上下文、会话和命令执行只是 Claude Code 这类产品的特殊需求。再换一个场景看看。

## 第三组对照：模型已经很强，环境却没有说清楚

OpenAI 做过一次内部实验：少量工程师主要依靠 Codex，在五个月内构建出约一百万行代码，创建并合并约 1500 个 Pull Request。单看最终数字，很容易把它理解成一次模型能力展示。〔5〕

但实验刚开始时，进展并不快。

OpenAI 在复盘中把早期瓶颈归到「环境定义不足」。模型会写代码，可它不知道仓库里哪些文档可信，不知道架构约束藏在哪里，也缺少方便查看页面、日志和运行状态的工具。完成一项修改以后，它甚至未必能独立判断结果好不好。

团队没有通过写一段更长的万能提示词解决这件事。他们逐渐补上 `AGENTS.md`、仓库内文档、页面与日志工具、自动测试、结构检查和可恢复的工作计划。模型没换，项目变得更容易被模型理解和操作，产出才开始加速。〔5〕

这组对照和 Claude Code 的事故很像。决定结果的变量不止模型智力，还有模型所处的工作环境。

OpenAI 把 CLI、IDE、Web 与桌面应用共用的核心直接称为「Codex harness」。Agent loop、线程持久化、配置与认证、沙箱中的工具执行、MCP、Skills 和审批，都由它负责。〔4〕

这并不是在替 GPT‑5.6 Sol 思考。它让模型的想法能够变成一个受约束、可观察的动作。

例如，模型决定运行测试，是推理；进程在哪个目录启动、能读到什么文件、退出码如何回传，是执行环境。模型认为改动已经完成，是判断；CI 是否通过、页面是否真的变化，是外部证据。把前一项做得再聪明，也不会自动产生后一项。

三组对照摆在一起，最初的猜想只对了一半。

## 猜想应该怎样修改

第一组对照说明，为旧模型补课的代码会迅速贬值。第二组说明，Agent 的表现能够在模型不变时被 Harness 改变。第三组又说明，即便模型足够强，项目也得把环境、约束和验收方式交代清楚。

因此，我会把开头的猜想改成这样：

> 模型会不断收回与推理有关的工作，但不会接管一个软件系统对外部世界承担的责任。

「与推理有关的工作」包括如何拆任务、选工具、反思结果、修补输出。今天写在 prompt 和工作流里的很多技巧，下一代模型也许直接就会。Bustamante 所说的苦涩教训，主要打中了这里。

外部责任是另一回事。

一次退款能否执行，取决于真实权限和审批状态；一次部署是否成功，要看服务端记录；一项代码修改是否合格，要由编译器、测试、浏览器或人来验收。这些事实不会因为模型更聪明就自动写进权重，也不该由正在完成任务的模型自行宣布。

Claude Code 与 Codex 在中间承担的是通用执行工作。应用开发者不必各自维护会话循环、Shell 调用和权限询问，这些能力逐渐沉到 Agent 产品里。站在使用者的项目中看，Harness 的确少了一些；站在整个系统中看，它只是换了位置。

项目还要留下自己的那一份。

`CLAUDE.md` 和 `AGENTS.md` 如果写的是「回答前先深呼吸」「遇到问题多反思」，生命周期可能很短。如果写的是仓库入口、测试命令、模块边界和完成标准，它们记录的是项目事实。以后承载这些事实的文件名可能会变，事实本身不会随模型版本消失。

沙箱和审批也一样。它们存在的原因不是模型听不懂警告。系统必须保证，即使模型误判、上下文丢失或遭到恶意输入，高风险操作仍然过不了边界。

Anthropic 在可信 Agent 的研究中，把模型、Harness、工具和环境分开讨论；Anthropic 与 OpenAI 的安全实践也都把权限控制放在模型外部。〔7〕 这并不神秘：判断可以交给智能系统，授权仍要由拥有责任的人和系统作出。

![模型负责推理，Claude Code 或 Codex 负责通用执行，项目环境提供规则、权限、状态和验收证据](/images/posts/harness-and-large-models/responsibility-map.webp)

## 下一次模型升级时，怎么检查自己的 Harness

与其争论 Harness 这个名词能活多久，我更愿意检查手里的每一段代码。

第一问：如果明天的模型聪明十倍，这段代码能不能直接删除？

如果答案是能，比如 JSON 修复、关键词路由、固定的反思模板，就把它写得简单一些。少做平台化，少绑定业务数据，给删除留条路。它的任务只是帮今天的模型过桥。

第二问：拿掉这段代码以后，系统还能证明动作被正确、合规地执行吗？

如果答案是不能，它大概就不属于临时补丁。权限检查、幂等状态、审计记录、测试结果和可观测工具，解决的都是现实世界的问题。模型越能自主行动，这些约束通常越值得认真对待。

还有一类代码会从项目里消失，却没有进入模型。比如会话管理、工具循环、通用沙箱和审批界面。看到它们被 Claude Code 或 Codex 接管时，可以放心删掉自己的实现，但不要误以为这项职责已经不存在。

这两个问题比「要不要做 Harness Engineering」更具体。它们关心代码因何存在，也允许答案随着模型和产品变化。

## 回到那句「死路」

「Scaffolding will not survive AGI」提醒开发者，不要把某一代模型的缺陷建成一座永久宫殿。这个提醒很有价值。

把它扩大成「Harness 最终都会在大模型上实现」，就解释不了 Claude Code 的上下文事故，也解释不了 Codex 实验里那些后来补上的项目环境。模型负责推理，系统仍得保存事实、执行动作、限制权限，并拿出结果。

未来，「Harness」这个词也许会过时。今天由应用开发者编写的许多认知补丁，也确实会被 Fable 5、Opus 5、GPT‑5.6 Sol 以及后续模型吸收。

不过，下次模型发布时，我不会先问 Harness 还活不活。我会重新做那三组对照：只换模型，删掉了什么；模型不换，外层系统改变了什么；模型和工具都有了，项目还缺什么。

答案不会永久有效。这个检查可以一直有效。

---

## 参考资料

1. [马克的技术工作坊：《Harness Engineering 到底是什么？概念、实战与争议，一次全部讲清楚》](https://www.bilibili.com/video/BV12LR1B3EUt/)；争议部分从 27:22 开始。另有 [YouTube 版本](https://www.youtube.com/watch?v=7nCzfgDjSo8)。
2. [Nicolas Bustamante：Hard Lessons Building Agents Since GPT-3.5（直达 The Bitter Lesson of Scaffolding 小节）](https://www.nicolasbustamante.com/blog/hard-lessons-building-agents-since-gpt-3-5#the-bitter-lesson-of-scaffolding)。
3. [Anthropic：An update on recent Claude Code quality reports](https://www.anthropic.com/engineering/april-23-postmortem)。
4. [Anthropic：How Claude remembers your project](https://docs.anthropic.com/zh-CN/docs/claude-code/memory)；[OpenAI：Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)；[Unlocking the Codex harness](https://openai.com/index/unlocking-the-codex-harness/)。
5. [OpenAI：Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)。
6. [Anthropic：Claude Fable 5 and Claude Mythos 5](https://www.anthropic.com/news/claude-fable-5-mythos-5)；[Claude Opus 5](https://www.anthropic.com/news/claude-opus-5)；[OpenAI：GPT‑5.6](https://openai.com/index/gpt-5-6/)。
7. [Anthropic：Trustworthy agents in practice](https://www.anthropic.com/research/trustworthy-agents)；[How we contain Claude across products](https://www.anthropic.com/engineering/how-we-contain-claude)；[OpenAI Codex：Agent approvals and security](https://learn.chatgpt.com/docs/agent-approvals-security)。
