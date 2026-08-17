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

>在浏览引擎层，我们初步判断将是可以进行比较大优化的一个方面。因为核心的爬取执行，以及LLM浏览点击网页灯各种界面的操作均在此层执行，而该层目前的核心逻辑较为简单且工具也比较有限。在这个部分，我们将梳理现在采用的 *反反爬虫* 操作以及页面提取方法。

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

解决完 *反反爬虫* 相关的问题，我们再来看一下 mini_skyvern 是如何从网页上提取相关元素并通过 Playwright 进行操作的。（~~写这部分单纯因为我缺乏这方面的基础 把相关知识用自己的语言组织并表述出来对于知识的理解将有较大的帮助~~）因为初步版本没有进行vlm增强，所以目前 mini_skyvern 还是通过 javascript 来提取网页上的元素（通过`Scraper.py`实现）并进行结构化处理。

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

**XPath**（XML Path Language）是一种标准的寻址语言，用于在DOM树当中唯一标识一个元素。在 mini_skyvern 当中，我们直接采取了一个从目标元素向根节点进行回溯的算法。该算法直接采用了 skyvern 项目当中的`Scraper.py`的 XPath 算法，时间复杂度是O(n)级别，后续优化的时候这或许也是一个能提升的点。 

![XPath](/assets/img/mini_skyvern/XPath.png){: width="100%" }

在此方法中，如果元素有id属性，直接采用id选择器并返回 `//*[@id="xxx"]`。（**id在DOM当中是唯一的**）

对于剩下的通用路径，则按照以下顺序进行暴力：
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

>以上内容即是相对来说比较值得记录复盘的一些内容，在写的过程当中我也注意到至少3处优化点可以持续尝试。其他的比如一些ActionType定义了哪些ActionType等问题其实只要清楚到底有哪些类型即可，就不展开了。
{: .prompt-warning}

本来还有缓存、存储以及前端三个部分应该写完，但是这部分的内容相对来说没有太多好分享记录的点，只要完成了 Agent 决策层和浏览引擎层两个层级的优化调整，剩下这三个层级部分直接让AI进行对应的修改调整即可。

>以上即是mini_skyvern初步实现的功能，接下来我们将结合成熟的 skyvern 进一步分析 mini_skyvern还能在哪些方面进行优化。

## part2 advancement from mini_skyvern to skyvern:


## part3 回顾总结

写在最后
>虽然写一篇技术文档来复盘做过的项目还是要花一定的时间，但是沉下心来认真比对和学习，确实能找到原本的小项目中明显需要优化一些的地方，并给agent的性能和使用感觉带来实打实的提升。那闲话少叙，改mini_skyvern去了。