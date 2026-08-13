---
title: mini_skyvern project
date: 2026-08-09 14:30:00 +0800
categories: [DEEP_AGENTS基础]
tags: [入门, AGENTS]
math: true
pin: true
mermaid: true
---

## part0 总揽：

**mini_skyvern project**是基于github上的`skyvern`仓库进行的一次相关功能复现，以便将相关的agent本地化部署。在AI 的协助下，这类任务的门槛变得比较低。一个没有足够专业知识的vibe coder 在了解清楚仓库架构(~~当然 这也可以借助AI 但是直接把仓库全丢给AI会很烧token 以当前部分LLM的token价格和容量而言还是非常奢侈的~~)相关协议以及agent功能的情况下，通过比较少的prompt就能够让LLM复现出skyvern的基本功能了。但是`skyvern`作为一个在github上有**22.7k stars**的仓库，如果是纯vibe coding堆出来的代码山的话，必然难以收获如此多的stars。

所以本篇文章将分成两大部分：
- 第一部分将针对**mini_skyvern project**这个复现出来的小agent进行工程化的复盘与拆解，加深对于项目的理解。
- 第二部分将研究探讨`skyvern`中难以通过vibe coding就实现，真正需要 *工程判断力* 的部分，观察工程师在编码层能带来的alpha增益。（~~虽然比较悲观的说 随着基础大模型的发展 这部分增益将会持续被侵蚀 但是从个人学习成长而言这件事情仍然有意义~~）

那么接下来，我们就正式开始吧！

![skyvern_title](/assets/img/mini_skyvern/skyvern_title.png){: width="100%" }

## part1 mini_skyvern project: