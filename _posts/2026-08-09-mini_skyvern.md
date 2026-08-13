---
title: mini_skyvern project
date: 2026-08-09 14:30:00 +0800
categories: [DEEP_AGENTS基础]
tags: [入门, AGENTS]
math: true
pin: true
mermaid: true
---

## part0 前言：

**mini_skyvern project**是基于 github 上的`skyvern`仓库进行的一次相关功能复现，以便将相关的agent本地化部署。在AI 的协助下，这类任务的门槛变得比较低。一个没有足够专业知识的 vibe coder 在了解清楚仓库架构、相关协议以及agent功能的情况下，通过比较少的prompt就能够让LLM复现出 skyvern 的基本功能了。但是`skyvern`作为一个在github上有**22.7k stars**的仓库，如果是纯 vibe coding 堆出来的代码山的话，必然难以收获如此多的stars。

所以本篇文章将分成两大部分：
- 第一部分将针对**mini_skyvern project**这个复现出来的小agent进行工程化的复盘与拆解，加深对于项目的理解。
- 第二部分将研究探讨`skyvern`中难以通过vibe coding就实现，真正需要 *工程判断力* 的部分，观察工程师在编码层能带来的alpha增益。（~~虽然比较悲观的说 随着基础大模型的发展 这部分增益将会持续被侵蚀 但是从个人学习成长而言这件事情仍然有意义~~）

那么接下来，我们就正式开始吧！

![skyvern_title](/assets/img/mini_skyvern/skyvern_title.png){: width="100%" }

## part1 mini_skyvern project:

>任务目标： 根据 Skyvern 仓库，做一个可以本地化部署、运行并调试优化的Agent demo。Agent将根据`Playwright`自动操控网页，完成类似点击、输入以及导航等工作；此外，Agent能够从网页当中提取结构化数据，例如扒取网页表格数据等要求；最后，对于Agent运行过程中存在的LLM决策过程进行记录，并将相应的过程缓存为可以复用的Python代码。
{: .prompt-info}

### 从Playwright开始

在正式学习阅读 skyvern 的仓库前，除了去 skyvern 的官网上体验了 skyvern 的功能和服务，我顺便也简单学习了一下 Playwright 这套由 Microsoft 团队开发出的工具。对于一个相关经验较为欠缺的小白而言，做复现之前先把底层的API工具所能做到的事情了解清楚对于后续的工作还是比较有帮助。

总体而言，Playwright 是由 Microsoft 开发的跨浏览器Web自动化与测试框架。通过提供统一的API来控制Chromium、Firefox 和 WebKit三大浏览器引擎，可用于端到端测试自动化、脚本化浏览器交互以及开发者工具集成等方面。此外，Playwright 还提供了相应的 MCP 协议以及专用的 CLI ， 使得其对于后续的Agent工程以及LLM驱动相关工具非常友好。

Playwright 在 github 上以 monorepo 的形式进行管理，下设多个npm的包。

在架构层面，用户代码通过客户端API与内部Channel协议通信， 再由服务端层驱动具体浏览器实现。

在 mini_skyvern project当中，我直接调用了 Playwright 的公共接口。目标是通过 vibe coding 在极短的时间内跑出demo，以便进行后续的调试。

![playwright](/assets/img/mini_skyvern/playwright.png){: width="100%" }

### mini_skyvern 框架总览

>在正式开始梳理**mini_skyvern project**的框架之前，可以先通过下图建立一个大体上的认识。

![gailan1](/assets/img/mini_skyvern/gailan1.png){: width="100%" }

整个框架下，**mini_skyvern** 包含5层核心架构层。

**首先是API层**。在API层，用户通过前端UI向RESTful API接口发出调度申请，由API层触发Agent层进行决策工作。在此基础上，加了CORS进行安全防护。

**其次是Agent决策层**。决策层的核心代码是`agent.py`，其负责保证每一次的quest严格遵循 observe -> Think -> Act -> Verify -> Extract 的顺序逻辑执行。其中，observe 观察的是 DOM 结构化的 json 文档（通过`scraper.py`实现）；然后，调用LLM模型进行具体决策（Think）；接着，根据 LLM 指令对浏览器引擎层下操作指令（Act）；随后检查操作页面是否发生改变（Verify）；最后把需要的数据拿出来（Extract）。

剩下的两个py文档中，`llm_client.py`将Agent Main Loop中的需求转化为结构化为给LLM的问题，（根据`prompts.py`当中定义了三类skills模版执行问询。三个模版一个是具体action的模版、一个是查询任务进度的模版、最后一个是针对性的提取数据的模版），然后再将LLM的回复解析成结构化的数据。同时，`llm_client.py`通过 `async` 定义对外方法，使系统在不同的 quest 之间异步运行的时候不会出现阻塞的情况。

**其三是浏览引擎层**。

**然后是缓存层**。

**最后是存储层**。

## part2 advancement from mini_skyvern to skyvern:


## part3 回顾总结

写在最后
>虽然写一篇技术文档来复盘做过的项目还是要花一定的时间，但是沉下心来认真比对和学习，确实能找到原本的小项目中明显需要优化一些的地方，并给agent的性能和使用感觉带来实打实的提升。那闲话少叙，改mini_skyvern去了。