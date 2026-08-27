---
title: mini_skyvern project（1）
date: 2026-08-09 14:30:00 +0800
categories: [DEEP_AGENTS基础]
tags: [入门, mini_skyvern project]
math: true
pin: false
mermaid: true
---

>由于整篇复盘的写作时间跨度较大，前面的内容和后面的内容之间的写作时间间隔将大半个月。在此期间，不断有新的收获，也意识到自己之前的见解相当片面且不成熟。因为整篇文章最终扩展的长度超乎预期，所以将原本的一篇复盘文章拆分成3个部分，并在原本写好的开头前，多加一个 part 来作为 mini_skyvern project 复盘的引言。
{: .prompt-warning}

## 写在所有内容之前：

**mini_skyvern project**是我首次尝试做一个中大型的项目，整个复现+复盘总结经历的时间跨度比较长，在此期间亦经历了一些方向上的调整和转变。

这也导致了整个**mini_skyvern project**的复盘相较于另外几篇而言，结构较为混乱。所以在文章最前面，我简单介绍一下三份复盘文章的内容和结构，以便读者进行阅读。

**mini_skyvern project（1）** 主要记录了对于单步quest的优化。其中包含了前期完成的工作，介绍前期做出的单步操作最小demo、基础的知识点、算法总结和初步调优思路；后期根据 deepwiki 以及 AI 多轮对抗审查后实际优化的方向和结果。

**mini_skyvern project（2）** 主要记录了整体架构和使用的优化调整。后期在重新使用体验了 skyvern 后，根据 deepwiki codemap 重新调整了 **mini_skyvern project** 的架构。整体的架构的架构发生了较大调整，所以单开一篇文章进行分享。

**mini_skyvern project（3）** 主要记录了后期的各种debug相关内容。专门将相关内容记录下来，以提高自己对于代码运行逻辑/技术优化等方面的敏锐度，进而提升自己未来喂 prompt 以及进行 Agent 工程的能力。

>如果想直接阅读剩余两篇文章，跳转链接在文章末尾

## part0 前言：

**mini_skyvern project**是基于 github 上的`skyvern`仓库进行的一次相关功能复现，以便将相关的agent本地化部署。在 AI 的协助下，这类任务的门槛变得比较低。一个没有足够专业知识的 vibe coder 在了解清楚仓库架构、相关协议以及agent功能的情况下，通过比较少的prompt就能够让LLM复现出 skyvern 的基本功能了。但是`skyvern`作为一个在github上有**22.7k stars**的仓库，如果是纯 vibe coding 堆出来的代码山的话，必然难以收获如此多的stars。（~~不过现在的github星星似乎膨胀的比较厉害 而且做完复现以后感觉也有可能就是AI堆出来的代码山doge~~）

所以本篇文章将分成两大部分：
- 第一部分将针对**mini_skyvern project**这个复现出来的小agent进行工程化的复盘与拆解，加深对于项目的理解。
- 第二部分将在第一部分**mini_skyvern project**的基础上继续优化，同时去探索那些通过vibe coding就实现，仍然需要人类去发挥价值的内容，观察工程师/产品经理/人在业务层以及编码层能带来的alpha增益。（~~虽然比较悲观的说 随着基础大模型的发展 这部分增益将会持续被侵蚀 但是从个人学习成长而言这件事情仍然有意义~~）

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

**其三是浏览引擎层**。通过调用 Playwright 的公共API，我们在浏览引擎层设置了13个函数作为操作工具，根据功能封装成4个py操作工具集。其中，`browser_manager.py`目的主要是绕过反爬程序、进行页面导航后获取page对象；`scraper.py`则进行页面爬取并截图；`action_handler.py`可以点击网页上的元素，输入字符等;`data_extractor.py`用来结构化数据的提取。

**然后是缓存层**。缓存层用来接收`agent.py`传来的成功且完整的action_history列表，这个列表当中记录着Agent循环中每一步的动作以及结果。随后`Script_generator.py`将整个action_history列表写成可执行的代码，并把相关代码逻辑保存到存储层的cached_scripts表。当下次同一个URL以及任务目标进来以后，可以直接调用缓存层的代码逻辑，为整体运行节省时间提升效率。

**最后是存储层**。结构比较简单，除了保存缓存的脚本表以外，还保存提取的数据表，步骤运行详情和任务主表等内容，以便让用户能够查询任务状态，Agent循环及最终的任务输出结果。

>至此，mini_skyvern project 的整体框架已经清晰。接下来将分析一下 vibe coding 中的技术实现细节，以期能够找到对于mini_skyvern进行优化的方向。

>接下来的分析中可能将着重讨论每个部分采用了哪些方法，还有哪些地方可以进行优化尝试进行研究，所以将不会把每个层内的所有内容全部面面俱到进行分析。(比如前端和缓存存储等部分 在mini_skyvern当中设计主要是围绕核心逻辑进行设计和服务 将被跳过)
{: .prompt-warning}

### Agent 决策层

>在 Agent 决策层，核心框架就是一个简单的链式结构，相关的设计与关注点主要集中在链式的执行结构和与LLM之间的交互上，并通过 prompt 工程对相关内容进行控制。

**先看链式结构的部分**。整个链条在`decide_actions()`执行具体操作，然后操作完马上进行`verify_completion()`。以此确保在经过决策的操作之后，新的页面状态是期望当中应该有的界面。如果操作失败，进行验证的LLM也能够判断是否继续进行尝试。

此外，在 vibe coding 的过程当中AI自动加入了动作解析容错（防止非法字段影响整个系统）和执行流程异常处理（确保任务结束的时候一定会browser.close()，避免出现进程泄漏）这两个步骤。

**对于LLM交互和Prompt工程部分**。`llm_client`封装了与LLM的交互（与OpenAI兼容），并且两级提取（如果API 原生JSON提取失败，则进行正则提取）确保能够提取出json。

由于 mini_skyvern 一开始设定的目标是让AI能够自己浏览网页并且根据网页的内容爬取一些需要的数据，所以在Prompt方面就设置了三个模版：一个是决策的模版，一个是验证的模版，还有一个则是进行数据提取的Prompt。

### 浏览引擎层

>在浏览引擎层，我们初步判断将是可以进行比较大优化的一个方面。因为核心的爬取执行，以及LLM浏览点击网页等各种界面的操作均在此层执行，而该层目前的核心逻辑较为简单且工具也比较有限。在这个部分，我们将梳理现在采用的 *反反爬虫* 操作以及页面提取方法。

>因为我对 *反反爬虫* 以及 *爬虫* 相关方法完全不了解，所以把AI推荐的方法基本上全部放了上去。这块的说明将直接让AI生成，快速了解即可✅。
{: .prompt-warning}

####  **反反爬虫**

以下的这段内容将直接由AI生成 （~~AI的表达方法还是比个人简洁清晰很多的 而且能结合大量的代码实例进行说明 这么看我就比较废了 笑~~）

mini_skyvern 总共采取了四层 *反反爬虫* 对策：

- **第一层：启动参数级隐藏**
    ```python
    args=['--disable-blink-features=AutomationControlled']
    ```
    这条Chromium启动参数会禁用 Blink 引擎的自动化控制标记，从浏览器内核层面移除`navigator.webdriver = true`的默认行为。

- **第二层：JavaScript注入级伪装**
    ```javascript
    // 在页面加载前注入，覆盖浏览器原生属性
    Object.defineProperty(navigator, 'webdriver', { get: () => undefined });
    window.chrome = { runtime: {} };
    Object.defineProperty(navigator, 'plugins', { get: () => [1, 2, 3, 4, 5]})
    ```
    这段代码在 `page.evaluate()` 中执行，在页面的 JavaScript 上下文启动之前就已经生效。它做了三件事：
    - 将 `navigator.webdriver` 重定义为 `undefined`（而不是 `true`）
    - 伪造 `window.chrome.runtime` 对象（正常 Chrome 浏览器都有这个对象）
    - 伪造 `navigator.plugins` 列表（headless 浏览器默认为空）

- **第三层：浏览器指纹伪装**
    ```python
    user_agent = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/120.0.0.0'
    locale = 'zh-CN'
    timezone_id = 'Asia/Shanghai'
    viewport = {"width": 1366, "height": 900}
    ```
    模拟一个真实 Windows 用户的 Chrome 120+ 浏览器，包括 User-Agent、语言、时区和视口尺寸。

- **第四层：行为级模拟**
    >（模仿真人的一些访问使用习惯）
    
    操作间隔模拟：
    ```python
    # action_handler.py
    delay = random.uniform(0.3, 1.2) # 每次动作前 300 - 1200ms 随机延迟
    await asyncio.sleep(delay)
    ```

    打字行为模拟：
    ```python
    # 不用 fill() (一次性填充)，而是逐字符 type（）
    await locator.focus()
    await locator.clear()
    for char in action.text:
        await locator.type(char, delay=random.uniform(50,150))
        # 每个字符间延迟 50-150ms
    ```
    
    为什么有效：
    - 反爬虫库（DataDome、 PerimeterX）分析键盘事件时序
    - fill()产生的时间几乎同时触发
    - 逐字符type()模拟真人打字模式

解决完 *反反爬虫* 相关的问题，我们再来看一下 mini_skyvern 是如何从网页上提取相关元素并通过 Playwright 进行操作的。（~~写这部分单纯因为我缺乏这方面的基础 把相关知识用自己的语言组织并表述出来对于知识的理解将有较大的帮助~~）因为初步版本没有进行模型视觉增强，所以目前 mini_skyvern 还是通过 javascript 来提取网页上的元素（通过`Scraper.py`实现）并进行结构化处理。

#### **DOM树**

![DOM_tree](/assets/img/mini_skyvern/DOM_tree.png){: width="100%" }

通过 javascript 提取下来的DOM树
{: style="text-align: center"}

取得DOM树以后，我们要将其转化为LLM可以理解的文本表示。通过转化后的文本，LLM需要理解三件事情：首先，哪些元素是可以做点击、输入以及选择等操作的；其次，需要通过XPath的方式，为后续的action操作定位元素位置；最后，需要提取整个页面的文本，供LLM理解上下文。（~~写着写着发现不对头了 我之前做的时候甚至没让ai帮我解决html结构混淆问题 感觉这个 *反反爬虫* 机制做了个寂寞啊~~）

在进行可交互元素提取的时候，我们要先进行选择器匹配。我让AI把CSS选择器覆盖的可交互元素类型调了出来并简单总结了一下，放在此处作为后续参考。（可能需要添加一些需要额外交互的元素以提升对于部分网站的识别处理能力）

```javascript
const selector = 'a, button, input, textarea, select, [role="button"], [onclick]';
const allElements = document.querySelector(selector);
```

目前覆盖了以下 Web 页面上的可交互元素类型：
- `a` —— 超链接（导航、跳转）
- `button` —— 按钮（提交、确认）
- `input` —— 输入框（文本、搜索、复选框）
- `textarea` —— 多行文本输入
- `select` —— 下拉选择菜单
- `[role="button"]` —— ARIA 角色为按钮的自定义元素（现代 SPA 框架常用）
- `[onclick]` —— 绑定了点击事件的元素

进行完选择器匹配以后，我们进行了可见性过滤 -- 因为解析出的DOM树上存在一些用户看不到的隐藏信息，而保留这些信息会对LLM识别整个网页的状况进行干扰。最终，设计的可见性标准为以下这几点：
- 元素要有实际的宽度以及高度(`rect.width>0 && rect.height>0`)
- CSS的 visibility 和 display 均已经打开

此外，最后还添加了一层交互元素数量限制。因为LLM的上下文窗口有限而且大多数网页可以进行交互的元素远低于80个（但是考虑到部分搜索页面上的元素会特别多），所以提取的交互元素的上限会限制在80个。至此，我们已经完成了截取DOM树并从中提取可交互的元素。我们接下来需要通过 XPath 的方式来为后续 action 操作定位元素位置。

**XPath**（XML Path Language）是一种标准的寻址语言，用于在DOM树当中定位一个元素。在 mini_skyvern 当中，我们直接采取了一个从目标元素向根节点进行回溯的方法。该算法直接采用了 skyvern 项目当中的`Scraper.py`的 XPath 生成策略，最坏情况下的时间复杂度是O(n)级别，而且对DOM树的结构变化非常敏感。后续优化的时候可以想办法在时间复杂度上和鲁棒性上进行优化。

![XPath](/assets/img/mini_skyvern/XPath.png){: width="100%" }

在此方法中，如果元素有id属性，直接采用id选择器并返回 `//*[@id="xxx"]`。（**id在DOM当中是唯一的**）

对于剩下的通用路径，则按照以下方法进行暴力：
   - 从当前元素开始，沿着 `parentElement` 链向根节点回溯
   - 在每一层级，计算当前元素在其**同名兄弟元素**中的位置索引（`index`）
   - 例如：如果当前元素是第 3 个 `<div>` 子节点，则索引为 3，生成 `div[3]`
   - 将每一级的路径片段推入数组 `parts`
   - 最终用 `/` 连接所有片段，得到完整路径

#### **元素信息标准化**

随后，我们对提取出来的每一个元素进行转换，将其变为一个标准化的JSON对象供LLM处理。

``` json
{
  "element_id": "e5",       
  "tag_name": "input",
  "text": "搜索...",
  "xpath": "/html[1]/body[1]/div[1]/form[1]/input[1]",
  "selector": "#search-box",
  "attributes": {
    "type": "text",
    "name": "q",
    "placeholder": "搜索...",
    "href": null,
    "role": null,
    "visible": true
  }
}
```

本来还有缓存、存储以及前端三个部分应该写完，但是这部分的内容相对来说没有太多好分享记录的点，只要完成了 Agent 决策层和浏览引擎层两个层级的设计，剩下这三个层级部分直接让AI根据功能需求完成剩下的代码即可。

>以上即是mini_skyvern初步实现的功能，我们不妨先初步总结一下可以进行优化调整的内容。
{: .prompt-tip}

### 潜在优化方向分析：

目前的 mini_skyvern 的核心决策层是一个agent-loop（遵循观察 - 思考 - 行动 - 验证 - 提取数据）循环。在这个循环当中的步骤中，嵌有基于playwright的工具，主要实现的逻辑是“ 提取DOM -> 生成XPath/CSS选择器”。我初步观察的问题主要在以下几点：
- 在做DOM -> LLM可读结构的转化时，只做了简单的DOM序列化正向处理，鲁棒性差。（skyvern在这个部分有大量细节处理）
- 在CSS选择器以及LLM可以操作的 Action 设置上过于简单。
- 缺乏视觉进行辅助处理，对网页上可视化组建的理解能力大幅下降。
- 面对页面更新或新页面的情况时，暴力求 XPath 的时候会频繁触发重排以及大量的节点遍历导致时间和空间上的效率不佳。
- 在进行 反反爬 以及验证码等问题处理时，设置较为简单，且对于一些动作执行循环的边界处理不清。

所以接下来我们将结合 deepwiki 解析 skyvern 仓库，完善 mini_skyvern 的功能并根据具体情境需求做进一步优化。

![间隔图](/assets/img/mini_skyvern/twomilinoyearslater.jpg){: width="100%" }

## part2 advancement from mini_skyvern to skyvern:

经过了一周多的调整，mini_skyvern 在单个 quest 上的表现已经远远优于一开始勉强能运行的状态。目前已经完成的测试任务包含从公司官网把公司管理层的相关信息爬取下来以及在东方财富网上搜索公司并爬取数据。虽然 mini_skyvern 目前距离实际应用场景还有一定的距离，但是考虑到项目本身的实验性质，目前这个效果算是阶段性成功了。未来如果要进行进一步调整优化的话，则仍需要进行更多的案例测试并持续完善整体的功能。

>暂时不会做进一步调整尝试,因为没有想到 mini_skyvern 有什么很好的实际应用场景。就算是正牌的skyvern我也认为从实用角度而言相对有限，但是从技术尝试和学习角度而言比较有意义。未来可以将相关经验迁移到别的agent项目上。
{: .prompt-warning}

接下来，将具体介绍一下更新调整的内容。

### Set-of-Mark 视觉标注

如果单独讨论 Agent 决策层，决策循环并没有发生改变，仍然遵循 *Observe -> Think -> Act -> Verify -> Extract* 这一决策循环。但是每一个决策步骤都进行了一定的过程调整。

在原`Observe`过程的设计当中， mini_skyvern 遵循了传统做法将DOM树序列化并发给LLM，但是真实网页的情况较为复杂。比如面对一个文本为空的节点，模型无法判断其到底是一个可点击的图标图标还是一个装饰；又或者当一个序列块当中有多个可交互内容的时候，仅凭序列化的DOM树无法处理。

因此，在决策循环中，在优化`Observe`过程提取文本DOM树的同时，增补了带编号蓝框标记的截图作为辅助识别。通过给每一个可交互元素一个编号`(e1,e2,······)`，并在截图上画出来，让模型同时看到带编号截图与同编号的元素树，形成图文互证。这既提升了对于前端网页结构的识别，还带标记的截图还能作为人类使用者核验 mini_skyvern 实际效果的参考，为debug提供帮助。

![东方财富](/assets/img/mini_skyvern/dongfangcaifuwang.png){: width="100%" }

带审阅画面效果如图所示
{: style="text-align: center"}

在具体实现上，AI大幅扩写了内嵌JS注入网页的代码文档(`annotation.py`)。流程上而言：
1. 程序会启动`isInteractable`通过标签白名单、cursor：pointer、tabindex以及角色属性等方式筛选出具体可交互的元素（筛选方式可以根据具体网站需求进行针对性调整/做成功能块）。
2. 通过`genUniqueld`这个函数给每个元素赋予一个DOM ID，即`(e1,e2,······)`，并记录包围成框（通过浏览器内置的API能够直接确认该元素的位置）。同时，这个ID还会同步保存到JSON结构体当中，里面同时包含从DOM当中提取下来的属性，从而提供关于该元素的补充语义。
3. 调用`drawBoundingBoxes`在页面上画出蓝色细边框以及编号标签（与元素树里的ID
保持一致），进而将视觉与结构完成对齐。
4. 如果模型在截图上看到了「e16 -> 搜索框」，然后LLM会根据对应的元素标记找到对应的结构体JSON，从而正确识别对应的元素。

由此，我们将模型理解网页页面由纯文本推断优化为图文互证，为后文debug提供了相应的“基础设施”。

>以后可能还会进一步调整 mini_skyvern，所以将annotation_JS.py当中的功能模块现有哪些功能保存在一个单独的markdown文档当中，具体查看可以点此链接查看。📎 [JS功能模块](/assets/docs/mini_skyvern1/checklist1.html)

### XPath 暴力重排问题

在优化暴力重排问题前，我们不妨先回顾一下XPath做元素定位的旧版工作方式。

```
1.序列话DOM计算XPath -> 把计算结果交给 LLM
2.LLM 去推断一个需要的 XPath -> 执行层用 document.evaluate(xpath)去定位元素
3.定位后读位置、点元素 -> getBoundingClientRect / click
4. 超出一定长度以后硬截断
```

其中前两步高度依靠网页的XPath以及LLM对元素位置的推断，但是因为XPath极易失效（比如执行了动作/加载了动态内容），所以在getBoundingClientRect之前极大概率需要重新计算整个页面的布局，导致需要进行大量的暴力重排计算。原方案为了控制整体的token成本直接进行了暴力截断，这也大大影响了最终的准确性。

所以在优化的过程当中，我们将XPath从主路径降级为兜底，在进行元素定位时优先尝试通过 Set-of-Mark 标注中设置好的 ID 进行配对（ID在结构块和属性里面 不受DOM树变化的影响），然后若无法定位到元素则降级为XPath计算处理，还是无法定位元素则用CSS选择器检查该类元素是否存在，没有则报错。

但是对于缓存一个元素定位识别的操作，保留的仍然是XPath。因为回放脚本运行的时候，标注引擎并没有在工作，所以仍然要依靠XPath。但是XPath是结合ID一起算出来并标记的，所以直接调用能同时保证效率与准确性。

### 多层降级链条（模型调用 + Token降级）

>相关代码在进行功能实现时清楚可以通过分级提升鲁棒性，并把需求提给 AI 即可。（自己可能要设置一下对应的 .env 文件来配置一下API接口）对于这一个小版块感觉没什么太多可以写的，所以直接让 AI 把代码实现中采用的分级策略总结一下贴过来了。
{: .prompt-warning}

首先是模型调用部分。

mini_skyvern 在模型调用层做了一件重要的设计：它不把"看图"和"做决策"绑死在同一个模型上，而是通过 .env 里的一个开关 MODEL_ROUTE 来切换整条调用链路。

**第一条路径采用 multimodel**。 这是最简单的方式——一个多模态模型（比如 Qwen3-VL）同时负责看截图和做决策。截图直接作为图片附件塞进决策请求里，模型一边看图一边输出"点击 e16"这样的动作。优点是链路短、延迟低，缺点是每次调用都要传图片，token 成本偏高。

**第二条路径采用 vlm_llm**。 这条路径把"看图"和"决策"拆成两个独立的调用。先让一个视觉语言模型（VLM）看截图，输出一段文字描述——比如"页面中间有一个搜索框，编号 e16，旁边有一个按钮，编号 e17"。然后这段文字描述作为"观察报告"，和 DOM 元素列表一起送给一个纯文本 LLM（比如 DeepSeek）来做决策。纯文本 LLM 不接收任何图片，所以它的 token 成本更低。

无论走哪条路径，系统都暴露三个调用入口给 Agent 主循环使用：decide_actions 负责每一步的决策，verify_completion 在动作执行后做二次校验确认任务是否真的完成了，describe_page 是路径二专属的"看图翻译"接口。为了应对模型端抽风（超时、5xx、配额耗尽），系统在四个层次上做了鲁棒性保障：
- **第一层是 SDK 级别**。每个 OpenAI 客户端在创建时就显式给了 120 秒超时和 2 次指数退避重试。如果不给这些参数，SDK 默认无超时——供应商一旦挂起，整个任务就永久卡死。
- **第二层是端点级别**。决策请求不是只打一个端点，而是走一条降级链。比如 vlm_llm 路由下，主力是 DeepSeek（付费），兜底是硅基流动的多模态模型（优惠券免费额度）。主力端点超时或返回 5xx 后，SDK 重试耗尽，错误冒泡上来触发切换到下一个端点。代码里明确注释了降级只能在"同类端点"间做——看图端（VLM）和决策端（LLM）职责不同，不能互相替代。
- **第三层是参数级别**。有些模型不支持 response_format=json_object 这个参数，调用时会返回 400 错误。遇到这种情况，代码会退回去用纯提示词约束（在 prompt 里写"请返回 JSON"），然后用正则表达式从回复里提取 JSON。
- **第四层是数据级别**。如果所有决策端点全部挂了，数据提取模块会跳过模型，直接从 DOM 里提取候选数据行返回，标记 degraded=True 并在 notes 里如实写明"字段未经模型清洗"。如果连这都失败了，Agent 主循环走 _fail 出口，把已经累积的数据抢救下来一起落盘。

另外，视觉侧有一个额外的降级契约：每一步系统都会把 visual_input_available 和 visual_degradation 两个字段写回页面快照里。如果本步没有截图（比如截图引擎注入失败），系统会在发给 LLM 的 prompt 顶部注入一个固定的前缀"视觉输入不可用"——让模型明确知道自己这一步是"瞎着"决策的，只靠 DOM 文本推断。这个信息不允许被静默吞掉，因为模型在不知道自己"看不见"的情况下做出的决策质量会严重下降。

![AI制图](/assets/img/mini_skyvern/AIzhitu1.jpg){: width="100%" }

模型调用框架
{: style="text-align: center"}

其次是Token 降级部分。

Agent 每步决策都要把大量信息拼成 prompt 发给模型——截图、元素列表、页面文本、操作历史。但模型的上下文窗口是有限的（比如 128K token），如果 prompt 超了就会报错。mini_skyvern 把这个问题从"事故"变成了"策略"——不是简单地截断，而是设计了一套逐级压缩的降级链。

**首先是估算口径**。文本按字符数除以 4 来近似 token 数（这是 GPT/Claude 系列模型的通用近似）。图片单独算——它不是按 base64 字符数算的，而是按视觉模型的固定计费档位来算：detail=low 每张约 85 token，detail=high 每张约 1500 token。这个分离是调试出来的教训：旧版用 len(json.dumps(messages))//4 估算，把一张 160KB 的 jpeg 图片的 base64 字符数算成了约 22843 个"token"——完全错误，因为图片的 token 成本跟字符数无关。

**然后是压缩链**。系统根据当前 prompt 的 token 数，决定使用哪个压缩等级：
- **Level 0**：完整内容，所有属性保留。只在 prompt 远未超预算时使用。
- **Level 1**：把长 href 截断到 200 字符，图片 src 截断到 100 字符，URL 的 query string 剥掉。这些对 LLM 决策没什么影响，但能省不少字符。
- **Level 2**：丢掉 SVG、path、circle 等非交互元素，去掉 style、class、ng-、mat- 等噪声属性。这些属性 LLM 根本用不上，但会占大量 token。
- **Level 3**：限制元素总数到 150 个。不是简单地取前 150 个，而是用关键词加权排序——从任务目标里提取关键词，包含这些关键词的元素优先保留。同时保证链接至少留 50 个（导航类任务的目标几乎总是链接，压到最后也不能把链接压没了）。
- **Level 4**：硬截断文本内容到原来的三分之二，操作历史只保留最近 1 条。

**最后还有一个硬顶**。不管前面怎么压缩，prompt 的 token 数绝对不能超过 PROMPT_HARD_CEILING_TOKENS（128000）。如果压到最后还超了，系统会按一个预设的优先级列表逐个丢弃字段——先丢操作历史摘要，再丢页面文本，再丢 VLM 观察报告。这些字段被丢弃后任务质量会下降，但至少不会报错。

此外，元素列表的序列化格式也做了优化。原来用 json.dumps(indent=2) 格式化输出，每个元素占好几行，非常占空间。现在改成了行式紧凑格式——只保留白名单属性（element_id、tag_name、text、type、placeholder 等 LLM 真正需要的），去掉 xpath、class、style 等无用信息。实测下来体积减少了约 77%。

![AI制图2](/assets/img/mini_skyvern/AIzhitu2.jpg){: width="100%" }

Token 降级框架
{: style="text-align: center"}

>以上内容全部由AI直接生成，可以看出质量确实比我自己写的质量要高很多（笑）
{: .prompt-danger}

### Loop_guard

Loop_guard 这部分内容其实更应该放在 mini_skyvern（3）里面写，因为是在debug过程中加入的一个py文件。但是为了保证内容的完整性，在 mini_skyvern (1) 当中也写一下。

在 mini_skyvern 的测试过程中，我们发现了这样一个问题：

>LLM 在「百度搜索」页面上进行输入关键词搜索的操作，却返回了不带 ID 的 input_text。动作失败，失败原因并没有回传。但是 LLM 以为关键词已经输入进去了，所以接下来4步持续去点击「百度一下」的按钮，但是搜索框始终是空的，页面没有任何变化，整个 quest_loop 陷入了空转的困境。
{: .prompt-tip}

加入`loop_guard.py`的目的主要就是为了解决 Agent 循环陷入无效空转的问题。（具体逻辑错误的诊断内容在 mini_skyvern（3）写Debug过程的时候应该会换一个角度再写一遍）

针对各种空转的风险，AI 设计了双层级的检测报错防线。

**在首层，先会对重复的动作进行检测。**即在每一步执行后，系统计算该步动作的状态。如果连续多步的状态完全相同，则将这个流程视为空转。同时，还设置了不同阈值来给出不同的反馈,比如连续2次相同就会给出警告，而连续4次相同的话则会强制终止整个流程。（这部分未来也是可以做定制化调整，以便有更强的适应性和针对性）

```python
### 状态块内含内容如下
def action_signature(action):
    return "|".join([
        action_type,   # 如 “click”
        element_id,    # 如 “e16”
        text[:40],     # 如 “搜索”
    ])
```

>后续调整的时候对scroll这个动作做了特判来决定动作是否一直在重复，因为在一个页面上多次重复滚动是正常的。

**在第二层，再对网页页面进行二次评估检测。**网页页面的状态衡量主要由三个指标组成，分别是URL、可交互元素 ID 序列以及正文长度（结构类信息）。如果多步的网络页面完全相同，则页面完全没有任何变化，表明 Agent 的操作失效。最后，与第一层设置了类似的阈值。连续3步页面不变就会出现警告，5步不变就终止进程。

### Actiontype 优化调整

补齐 Actiontype 的方式相对简单，基本上将 skyvern 仓库当中有的 Action 直接搬运了过来。此外，还添加了`add_pre_hook/add_post_hook`两个文件，针对LLM在网页上的操作进行了监察与限制。

比如审计Hook能够记录Agent每一步都做了什么，进而形成可以追溯、调试与排查的操作日志。而限速能够限制AI的操作频率；Human-in-the-loop则让AI在执行高风险动作之前暂停，等人类确认后才继续进行下一步操作。

具体的 Actiontype 如下图所示。

| 维度 | 旧版（7 种动作） | 新版（19 种动作） |
|------|:---:|:---:|
| 单页点击/输入 | ✅ | ✅ |
| 页面导航（前进/后退/刷新） | ❌ | ✅ |
| 多标签页操作 | ❌ | ✅ |
| 无限滚动/懒加载 | ❌ | ✅ |
| 文件上传/下载 | ❌ | ✅ |
| 验证码处理 | ❌ | ✅（识别+上报） |
| 自定义 JS 逃生 | ❌ | ✅ |
| 复选框状态感知 | ❌ | ✅ |
| 操作跳过 | ❌ | ✅ |
| 横切扩展（hooks） | ❌ | ✅ |

### 数据提取算法优化

在 Set-of-Mark 视觉标注板块，我们介绍了 JS + Playwright 注入网页然后进行元素标记以及对应的操作。但是将标记好的网页面板只是最基础的样式，要让LLM进一步根据网页内容做决策，需要我们进一步加工标记好的网页面板。所以在 DOM 处理与数据提取优化这个板块，我们就将讨论如何这个问题。

#### 元素收集

在旧版的 mini_skyvern 当中，元素收集采用了简单的遍历方法，即按照文档序进行遍历。而百度的源码结构是导航栏等功能性栏目在前，而正文等重要内容在后。所以在LLM读到正文前，结构性的框架就要占掉非常多的token消耗。而我们在一开始设计 mini_skyvern 喂 prompt 的过程中要求卡住token消耗量，所以 AI 在实现的时候在`collect(maxElements)`中卡住了上限，导致LLM完全没有拿到有价值的信息。

为了让LLM尽可能拿到核心的"正文"内容，最终我们尝试了以下优化。

**首先，是将html规范写死的各种交互元素优先级放到其他元素之前**。即对于`<a>、<button>`这类确定的可交互元素，优先级将排在`<div>`、`<span>`这类元素之前。（但是本步骤对于同样的`<a>`将不区分正文和索引框之间的优先级顺序）

具体代码实现如下
```javascript
// 第一轮：只走核心交互标签
for (const el of document.querySelectorAll("a, button, input, select, textarea")) {
    if (results.length >= limit) break;
    tryCollect(el);
}
// 第二轮：按文档序补齐其余
for (const el of document.querySelectorAll("*")) {
    if (results.length >= limit) break;
    tryCollect(el);
}
```

**其次，剔除布局容器等干扰交互目标选择的内容**。进行相关处理的原因是在百度上进行网页搜索的过程中发现存在内容标注边框标记内容过大，无法选取具体交互元素，所以进行了处理。（搜索引擎或多或少有类似处理需求）

判别标准简单粗暴，就是一次乘法比较：**元素包围盒面积 ÷ 视口面积 > 30% 直接拒收**。原因则是因为真正的交互目标不可能盖住四分之一屏，盖那么大的几乎必然是布局容器。此外，这个阈值可以根据不同搜索引擎的具体需求进行调整，做定制化的设计。

**接着，是要进行父子去重**。这部分调整和百度搜索条目的设置有关，可以先根据链接当中的说明文档再了解具体操作。📎 [搜索条目设置](/assets/docs/mini_skyvern1/sousuotiaomu.html)

而在元素收集的时候，进行父块子块去重还是考虑到token限制的问题。百度通过父子块设计的目的是为了提升浏览器用户的使用体验，但是本质上父子指向的网址是同一个网址。把这样的内容直接塞入LLM会导致紧张的token限额被重复上下文所挤占，而且后续LLM在读取选择交互元素的时候还需要再画时间进行决断，影响效率。（最极端的情况下甚至会有无法交互的父块，然后导致程序运行卡死）

所以在满足 **（父子俩都在收集列表里）&&（（子块被父块盖住80%）||（子 ∈ {a, button} 且 父 ∈ {div, span, li, p, section, article}））** 的情况下就会把删除父块保证准确性与效率。

#### 列表选择器：全打分择优



#### 跨页积累与内容指纹

#### 缓存脚本

缓存脚本相关的内容在文章前面的部分有涉及过，在此处再进行一个整体总结。

首先，在XPath 烧进脚本任务成功后，`ScriptGenerator` 把动作轨迹固化成一份独立的 Playwright 脚本。定位用的是每个动作执行前由 agent 用 ID 反查出来的真实 XPath，因此脚本脱离标注引擎也能跑 —— 回放时页面上没有ID，XPath 是唯一定位载体。相同任务再来时命中缓存直接回放，极大提升效率

4.2 提取钩子：回放路径与 Agent 路径共用同一个累积器关键设计：回放不是"只跑动作不提取"。生成器在脚本的三个位置注入`await _extract_page(page)` 钩子：1. **起始 `goto` 之后**（第一页的数据不丢）；2. **每个翻页动作之后**（翻页是内容变化的确定性时刻）；3. **结尾**（兜底）。`ScriptRunner.run_on_page(on_step=...)` 把提取函数注入脚本命名空间；旧缓存脚本不引用钩子时无害（no-op）。`_run_with_code` 用的正是与 Agent路径**同一个** `ExtractionAccumulator`，所以去重、预算、核对、落库全套语义完全一致——无论走模型决策还是走脚本回放，用户拿到的数据格式与完整度承诺是同一套。


### 反反爬与边界处理

新版的 反反爬虫策略 仍然是按层次实施（但并不是像降级策略那样遵循一定的严格顺序），遵循 浏览器伪装 -> 导航两段式goto -> 动作级拟人 -> 策略插件`antibot.py` 这样的流程。其中，浏览器伪装、goto和动作级拟人在一开始的设置当中就是有的，在新版中部分做了优化。

#### 浏览器伪装

在浏览器伪装方面，进行了伪装项的增补。通过一个表格简单确认一下条目即可。

| 伪装项                  | 真浏览器的值       | Headless 的破绽                 |
| ----------------------- | ------------------ | ------------------------------- |
| `navigator.webdriver`   | `undefined`        | `true`                          |
| `window.chrome`         | 存在               | 不存在                          |
| `navigator.plugins`     | 3 个插件           | 空数组                          |
| WebGL 渲染器            | Intel Iris...      | `"SwiftShader"`（一眼假）       |
| `hardwareConcurrency`   | 4~16               | 常为 1 或缺失                   |

#### 导航两段式goto

>对于像我一样不是很了解什么是 goto(浏览器加载) 的初学者，我在这里放一个 AI 编写的小文档，可以通过这个文档先了解一下相关知识再阅读这个章节。📎 [goto简介](/assets/docs/mini_skyvern1/goto.html)

在旧版的 mini_skyvern 当中，面对打不开的主机（有些主机通过这种方式来反爬），goto（domcontentloaded -> load -> commit）这个流程会直接跑满超时。

所以在新版 mini_skyvern 当中，多设置了一个`navigate()`函数来优化： 只有在确认主机是可响应的情况下才会等待整个goto流程运行完。
```python
# 第一段：可达性探针 -- 只等“响应头”到达
await page.goto(target, wait_until="commit", timeout=settings.NAV_PROBE_TIMEOUT_MS)
# 第二段：主机已确认活着，尽力等 DOM 就绪，等不到不判失败
for state in ("domcontentloaded", "load"):
    try:
        await page.wait_for_load_state(state, timeout=settings.NAV_SETTLE_TIMEOUT_MS)
        return
    except Exception:
        continue
```
如果在主机运行过程中直接出现了域名报错、网络不通等情况，那么直都不用触发到响应头，失败后会直接报错“请检查网址是否正确”等，方便进行检查。

而对于反应比较慢的主机（或者有反爬load功能的），旧版 mini_skyvern 会一直被挂着然后卡死；新版会等待，但是也添加了等待时间上限的cap，防止无限等下去。

总体而言，这部分主要是提升整体的运行效率。

#### 动作级拟人

这部分相较于旧版没有什么额外的变化，基础的动作级拟人维持了原样（够用了）。如果后续有针对特定网站要进行 反反爬 则通过挂载的策略插件`antibot.py`进行定制化特殊处理。

#### 策略插件`antibot.py`

`antibot.py`内部设计了一个可编辑调整的策略注册表，一开始内置了三条策略基础策略，如同意验证弹窗等。除了内置的策略意外，还可以通过挂载的方式添加新的具体 反反爬策略。

目前来看，除了输入验证码这类依靠视觉模型协同的实用 反反爬 策略还没有接入 mini_skyvern 以外（但后续也可以在`antibot.py`中专门设计一个功能），对于反爬功能较弱的网站基本上都能实现正常浏览以及基础的数据提取操作了。

## part3 reflection：

写到这里，mini_skyvern（1）差不多就写完了。在做这个项目时，最大的感受大概是在做一个任务/项目的时候不要老想着跟上AI的脚步，不要认为自己能在执行一个小的task的过程中与AI协同就能提高最终的整体表现。AI在一个项目上的表现基本上取决于你在开始任务前所掌握的知识水平以及技术构想能力。一边做一边学是一种效率极低的学习方式，且难以达成原本设定的目标。

就我个人的体验而言，我们需要做的是在任务开始的时候尽量设计一个优秀的框架，将需求用技术性和实用性的语言表达出来；在任务中面对AI在大范围铺开进行尝试的时候提供一些人类的小直觉来节省整体token数量的消耗；以及在任务结束后，根据具体的使用感受给AI明确的反馈。（仍然是要考虑技术层面以及实用层面）学习和复盘的任务放在任务已经迭代完成后进行，将AI在实际生产中产生的知识和内容内化，以提升个人在下一轮实操中的表现。

最近有学者提出过这样一个观点，大意是AI在问题的定义、结果的评价方面的“品味”能力已经超过人类，并对此感到悲观。或许AI的智力水平从各种意义上超越人类已经成为一种无法避免的事实，但是这其实对人类的学习水平提出了更高的要求。一方面，非人智能产生的知识与技术并不属于人类，即使AI已经跑在了人类之前我们仍然有必要将AI创造的各种知识内化到人类自己的知识系统当中并传授给下一代；另一方面而言，主观能动的权利不出意外应该会长期掌握在人类手中。既然人类仍然需要去扮演驱动者的角色，我们应学会基于技术/现实去提出合理的想象，提出符合人类价值的需求。这些都鼓励着我们，不断去学习，不断去进步。

![qianduanyemian2](/assets/img/mini_skyvern/yemianjietu.png){: width="100%" }

项目前端页面
{: style="text-align: center"}

>如果想要了解框架上的优化调整，请前往 mini_skyvern（2）

[下一篇：mini_skyvern（2）]({% post_url 2026-08-21-mini_skyvern2 %})

>如果想要了解迭代过程中debug相关的内容，请前往 mini_skyver（3）

[下一篇：mini_skyvern（3）]({% post_url 2026-08-24-mini_skyvern3 %})