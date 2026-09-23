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

### Welcome to the start!

>在浏览 mini_skyvern 系列文章之前，可以从这篇文章开始。虽然这是写作顺序上的第二篇文章，但阅读这篇概览能帮助读者建立对于 项目的整体认知，从而更好地理解其他文章的内容，遂放在推荐阅读顺序的第一篇。

在 **mini_skyvern project 技术复盘** 中，我们详细介绍了单步 quest 从一开始的 demo -> final version的演化过程。但是光有单步 quest 的 mini_skyvern 在功能上以及可玩性过于单一，所以在后期的调整中，mini_skyvern 参考 skyvern 加入了 的多 quest 组合功能等，完善了整体的架构。在 **mini_skyvern project 架构简览**当中，我们将详细介绍 mini_skyvern 最终版本的功能以及架构相关方面的内容，建立对于终版 mini_skyvern 宏观上的认识。

## 功能架构

![框架说明图](/assets/img/mini_skyvern2/mini_skyvern2shuomingtu.jpg){: width="100%" }

## 技术架构

## 代码架构

在代码架构方面，我们将主要参考 AI 生成的图表和文档。（作为技术架构的辅助参考）

```
mini_skyvern/
├── main.py                  服务入口：启动 FastAPI，监听 8000 端口
├── config.py                全局配置：路径、超时、阈值、模型路由
│
├── api/                     ── ① 接入层
│   ├── server.py            FastAPI 主应用：任务/数据/调试端点、前端页面
│   ├── task_input.py        任务输入规范化与提取目标建议
│   ├── deps.py              数据库依赖注入
│   └── routers/
│       ├── workflows.py     工作流编排端点
│       └── automation.py    自动化仓库端点（任务模板管理）
│
├── agent/                   ── ② 决策层
│   ├── agent.py             Agent 主循环：感知→提取→决策→执行的心脏
│   ├── prompts.py           各场景提示词模板
│   ├── llm_client.py        模型客户端：超时、重试、端点降级链
│   ├── token_budget.py      Token 预算管理：配额加权、分层降级
│   ├── loop_guard.py        循环守卫：三道熔断防线
│   └── account_balance.py   模型账户余额查询
│
├── browser/                 ── ③④ 感知层 + 动作层
│   ├── annotation_js.py     核心：元素收集五道筛子 + 标注画框
│   │                        （以字符串形式存放、注入浏览器执行的 JS）
│   ├── scraper.py           页面抓取调度：串联收集与截图
│   ├── screenshot.py        截图引擎：标注截图 + 审阅帧
│   ├── change_detector.py   布局指纹变化检测：审阅帧稀疏抽样
│   ├── action_handler.py    动作执行：点击/输入/翻页 + 拟人节奏
│   │                        + 验证码探测
│   ├── browser_manager.py   浏览器生命周期：指纹伪装 + 两段式导航
│   ├── data_extractor.py    提取流水线：选容器打分 + 累积器
│   └── fast_extractor.py    快速提取：跳过动作循环的抓取
│
├── cache/                   ── 路径 B 回放机制
│   ├── script_generator.py  把成功动作序列固化为脚本（含提取钩子）
│   ├── script_runner.py     回放执行：注入提取函数进脚本命名空间
│   └── （生成的脚本存放在 generated_scripts/ 目录）
│
├── automation/              ── 反爬策略与动作扩展
│   ├── antibot.py           反爬策略注册表与三个内置策略
│   ├── base.py / registry.py 策略基类与注册机制
│   ├── loader.py            动作类型加载器
│   └── builtin/             内置动作：反爬施加、文件下载、数据抓取
│
├── workflow/                ── 工作流编排
│   ├── engine.py            工作流执行引擎
│   ├── block_runner.py      单块执行：导航块/提取块/数据块
│   ├── context.py           块间数据传递上下文
│   ├── frame_publisher.py   审阅帧发布（配合变化检测）
│   └── url_resolver.py      块间网址变量解析
│
├── storage/                 ── ⑦ 存储层
│   ├── database.py          SQLite 连接管理
│   ├── models.py            数据模型：任务、步骤、数据、脚本
│   └── repositories/        仓储层：任务/工作流/自动化仓库读写
│
├── frontend/                ── 用户界面
│   ├── index.html           单任务界面
│   ├── workflow.html        工作流编排界面
│   └── static/js/           监控轮询、块编辑、数据展示逻辑
│
└── demo_site/               验收演示站点（结构固定，可复现）
```

## 小结

至此，我们已经介绍完