---
title: "mini_skyvern project 架构简览"
date: 2026-08-21 14:30:00 +0800
categories: [DEEP_AGENTS基础]
tags: [入门, mini_skyvern project]
math: true
pin: false
mermaid: true
---

> mini_skyvern系列文章推荐阅读顺序：**mini_skyvern 架构简览** -> **mini_skyvern 技术复盘** -> **mini_skyvern project Debug Case**
{: .prompt-tip}

>如果想要了解项目技术实现细节，请前往 mini_skyvern project 技术复盘

[下一篇：mini_skyvern project 技术复盘]({% post_url 2026-08-09-mini_skyvern %})

>如果想要了解迭代过程中debug相关的内容，请前往 mini_skyvern Debug Case

[下一篇：mini_skyvern Debug Case]({% post_url 2026-08-24-mini_skyvern3 %})


>本篇文章将介绍 **mini_skyvern project** 的结构调整。在 **mini_skyvern project 技术复盘** 中，我们详细介绍了单步 quest 从一开始的 demo -> final version的演化过程。但是光有单步 quest 的 mini_skyvern 在功能上以及可玩性过于单一，所以在后期的调整中，mini_skyvern 参考 skyvern 加入了 的多 quest 组合功能等，完善了整体的架构。在 **mini_skyvern project 架构简览**当中，我们将详细介绍功能以及架构相关方面的内容，建立对于 mini_skyvern 宏观上的认识。


开始前，先放一张框架图进行直观展示。

![框架说明图](/assets/img/mini_skyvern2/mini_skyvern2shuomingtu.jpg){: width="100%" }