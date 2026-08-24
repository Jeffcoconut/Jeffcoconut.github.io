---
layout: page
---

>直接AI生成的，内容清晰但有点丑

# annotation_js.py 内嵌 JS 操作与功能模块详解

> 源文件：`mini_skyvern-demo/mini_skyvern/browser/annotation_js.py`
>
> 该 Python 文件的核心是一个约 1180 行的 JavaScript 字符串 `ANNOTATION_JS`，通过 Playwright 的 `page.evaluate()` 注入浏览器页面执行。注入后在 `window.__msk` 上挂出一组方法，Python 侧通过调用这些方法完成所有浏览器端操作——Python 自身不做任何 DOM 解析。

---

## 一、整体架构概览

整个内嵌 JS 可拆分为 **12 个功能模块**，按数据流顺序排列如下：

| 序号 | 模块名称 | 代码行范围 | 核心职责 |
|:---:|---------|:---------:|---------|
| 1 | 矩形运算与可视区裁剪 | 23–108 | 计算元素在屏幕上的位置与大小，提供矩形裁剪、相交判定、缓存机制 |
| 2 | 可见性判定 | 110–140 | 判定元素是否对用户可见（排除 display:none、opacity:0 等隐藏元素） |
| 3 | :hover 指针样式推断 | 142–187 | 扫描样式表找出"hover 时 cursor:pointer"的选择器，弥补 Chrome 无法运行时读取 :hover 样式的限制 |
| 4 | CSS 反推 disabled / required | 189–224 | 通过 CSS class 和伪元素推断前端框架用样式表达的禁用/必填状态 |
| 5 | 特殊控件识别 | 226–338 | 识别 Select2、React-Select、Angular Material 等各类前端框架的下拉框和日期选择器 |
| 6 | isInteractable 可交互性判定 | 349–413 | 六层信号链综合判定元素是否可交互（标签白名单→ARIA→事件→tabindex→cursor→框架控件） |
| 7 | 元素收集与去重 | 454–566 | 两轮收集可交互元素、分配 unique_id、父子去重、面积过滤 |
| 8 | XPath 与文本提取 | 415–452 | 计算元素 XPath 路径、提取元素文本（含伪元素内容） |
| 9 | 四叉树与视觉分组 | 660–791 | 用四叉树把视觉位置重叠的元素归为一组，避免截图上画大量重叠蓝框 |
| 10 | 画框与移除框 | 793–872 | 为每个分组画蓝色边框和 ID 标签（Set-of-Mark 标注），供 LLM 截图决策 |
| 11 | 滚动控制与页面清理 | 874–924 | 滚动翻页、获取滚动状态、提取页面正文、清理所有 unique_id |
| 12 | MutationObserver 增量抓取 | 926–1167 | 监听 DOM 变化，只抓取因用户操作而新增的可交互元素（如下拉菜单弹出） |

---

## 二、各模块详细说明

### 模块 1：矩形运算与可视区裁剪（第 23–108 行）

这个模块是所有后续几何计算的基础。每个可交互元素都需要先算出它在屏幕上的位置和大小，才能进行画框、分组、面积过滤等操作。

| 函数/对象 | 功能 | 说明 |
|----------|------|------|
| `Rect.create(x1, y1, x2, y2)` | 创建矩形对象 | 包含 left/top/right/bottom/width/height 六个属性 |
| `Rect.copy(r)` | 复制矩形 | 防止引用被意外修改 |
| `Rect.intersects(a, b)` | 判断两个矩形是否相交 | 用于后续视觉分组 |
| `cropRectToVisible(rect)` | 将矩形裁剪到视口范围内 | 完全在视口外或裁剪后宽高 < 3px 则返回 null |
| `getVisibleClientRect(element, testChildren)` | 获取元素的可见矩形 | 核心函数。零尺寸父元素会尝试用浮动/绝对定位子元素的矩形代替 |
| `rectCache`（WeakMap） | 矩形缓存 | 避免同一帧内重复计算；滚动后由 `clearRectCache()` 清空 |

**设计要点**：`getVisibleClientRect` 处理了一个常见场景——某些容器元素自身宽高为 0，但内部的浮动或绝对定位子元素是可见的。此时递归检查子元素来获得真实的可见区域。

---

### 模块 2：可见性判定（第 110–140 行）

在判定元素是否可交互之前，先要确认它是用户能看见的。

| 函数 | 功能 | 判定逻辑 |
|------|------|---------|
| `isHidden(element)` | 检测元素是否隐藏 | `display: none` 或 `hidden` 属性 → 隐藏。**特例**：`<input type="submit/button">` 带 `hidden` 但 `cursor: pointer` 的不算隐藏 |
| `isVisible(element)` | 综合可见性判定 | `isHidden` + `visibility` + `opacity` + `checkVisibility()` API，任一不可见即返回 false |
| `isScriptOrStyle(element)` | 过滤非视觉元素 | `<script>`、`<style>`、`<noscript>`、`<template>` 标签直接排除 |

---

### 模块 3：:hover 指针样式推断（第 142–187 行）

Chrome 无法在运行时直接读取 `:hover` 伪类的计算样式。这个模块通过扫描样式表来弥补这个限制。

| 函数 | 功能 | 说明 |
|------|------|------|
| `hoverPointerSelector()` | 扫描所有样式表，提取 hover 时 cursor:pointer 的基础选择器 | 遍历 `document.styleSheets` → 找出含 `:hover` 且 `cursor: pointer` 的 CSS 规则 → 去掉 `:hover` 部分保留基础选择器 → 合并成一条大选择器。跨域样式表读取失败时直接跳过。结果缓存在 `window.__mskHoverSelector` 上，上限 400 个选择器 |
| `hasHoverPointer(element)` | 判断元素是否匹配上述选择器 | 用 `element.matches()` 一次调用即可判定，避免逐个元素计算 hover 样式 |

**为什么需要**：很多网站用 CSS `:hover { cursor: pointer }` 来表示元素可点击，但不绑定 `onclick` 等事件属性。这是 `isInteractable` 的第 ⑤ 层信号源。

---

### 模块 4：CSS 反推 disabled / required（第 189–224 行）

现代前端框架（React、Vue 等）往往不用 DOM 属性标记禁用/必填状态，而是用 CSS class 来表达。仅靠 `element.disabled` / `element.required` 会漏掉这类元素。

| 函数 | 功能 | 检测方式 |
|------|------|---------|
| `checkStringIncludeRequire(str)` | 检查字符串是否含必填标记 | 是否包含 `*`、`✱`（U+2731）或 `require` |
| `checkRequiredFromStyle(element)` | CSS 反推必填状态 | ① `::after` 伪元素的 `content` 是否含必填标记（如红色 `*` 号）；② `className` 是否含 `require` 关键词 |
| `checkDisabledFromStyle(element)` | CSS 反推禁用状态 | ① `className` 含 `disabled`；② `opacity ≤ 0.4` + `pointer-events: none` 组合（框架常用的禁用视觉样式） |

---

### 模块 5：特殊控件识别（第 226–338 行）

各类前端框架实现的下拉框、日期选择器在 DOM 上往往只是普通的 `<div>` 或 `<input readonly>`，需要结合 `role` / `aria-*` / `className` 组合判定。

| 函数 | 识别目标 | 核心判定线索 |
|------|---------|------------|
| `isComboboxDropdown(element)` | 原生 combobox 输入框 | `<input>` + `role` + `aria-haspopup` + `aria-controls` + `readonly` |
| `isDivComboboxDropdown(element)` | div 模拟的 combobox | `<div>` + `role="combobox"` + `aria-controls` + `aria-haspopup` |
| `isDropdownButton(element)` | 下拉按钮 | `<button type="button">` + `aria-haspopup="listbox"` 或 `aria-expanded` |
| `isSelect2Dropdown(element)` | Select2 组件 | `<a>` 的 className 含 `select2-choice`，或 `<span>` 的 className 含 `select2-selection` + `role="combobox"` |
| `isSelect2MultiChoice(element)` | Select2 多选输入 | `<input>` 的 className 含 `select2-input` |
| `isReactSelectDropdown(element)` | React-Select 组件 | `<input>` 的 className 含 `select__input` + `role="combobox"` |
| `isReadonlyInputDropdown(element)` | 自定义只读下拉 | `<input>` 的 className 含 `custom-select` + `readonly` |
| `hasNgAttribute(element)` | Angular 属性检测 | 元素是否有任何 `ng-*` 开头的属性 |
| `isAngularDropdown(element)` | Angular 下拉组件 | 有 `ng-*` 属性 + `aria-label` 含 `select` 或 `choose` |
| `isAngularMaterial(element)` | Angular Material 检测 | 元素是否有任何 `mat-*` 开头的属性 |
| `isAngularMaterialDatePicker(element)` | Angular Material 日期选择器 | 有 `mat-*` 属性 + `<input>` + 在 `mat-datepicker` 或 `mat-formio-date` 内 |
| `isDatePickerSelector(element)` | 通用日期选择器 | className 含 `datepicker` / `date-picker` / `datetimepicker` |
| `isSelectableWidget(element)` | **综合入口** | 以上所有函数 + 原生 `<select>` 的并集，供 `snapshot` 标记 `is_selectable` |

---

### 模块 6：isInteractable 可交互性判定（第 349–413 行）

这是整个标注引擎的核心过滤器。它要回答一个问题：**这个元素，用户能不能跟它交互？**

采用六层信号链，按优先级从高到低依次检查：

| 优先级 | 信号类型 | 检查内容 | 可靠性 |
|:-----:|---------|---------|:------:|
| ① | 标签白名单 | `<a>`（有 href）、`<button>`、`<select>`、`<textarea>`、`<input>`（非 hidden）、`<label>`（有 control） | 最强 |
| ② | ARIA 角色 | `role` 属性在 WIDGET_ROLES 集合中（button/link/checkbox/radio/combobox/tab/switch/slider 等 17 种） | 强 |
| ③ | 事件属性 | 元素上有 `onclick`、`jsaction`（Google）、`ng-click`（Angular）、`(click)`（Angular）、`@click`/`v-on:click`（Vue） | 强 |
| ④ | tabindex | `tabindex="0"` 表示键盘可达 | 中（可能误判布局容器） |
| ⑤ | cursor:pointer | 对特定标签（div/span/li/img 等 16 种）检查计算样式或 :hover 推断的 cursor:pointer | 中弱 |
| ⑥ | 框架控件兜底 | 调用模块 5 的各特殊控件识别函数 | 中 |

**前置过滤**：在进入信号链之前，先排除 `<script>/<style>` 等标签、`<html>/<body>/<head>`、不可见元素、`pointer-events: none`（但 disabled 元素仍保留——模型需要知道按钮是灰的）。

---

### 模块 7：元素收集与去重（第 454–566 行）

通过 `isInteractable` 判定后，需要把元素收集起来并做后处理。

| 函数 | 功能 | 说明 |
|------|------|------|
| `genUniqueId()` | 生成唯一 ID | 格式为 `e1`、`e2`、`e3`...，全局递增计数器，写回 DOM 的 `unique_id` 属性 |
| `getSelectOptions(element)` | 采集 `<select>` 的选项 | 返回所有 option 的 {index, text, value} 和当前选中值 |
| `snapshot(el)` | **单个元素的完整快照** | 分配 unique_id → 采集 DOM 属性（type/name/placeholder/href/role/value/disabled/required/checked 等）→ 处理 Shadow DOM host 映射 → 采集 select options → 标记 isSelectable → 计算 XPath |
| `collect(maxElements)` | **两轮收集 + 过滤** | 第一轮只收 `<a>/<button>/<input>/<select>/<textarea>`（保障核心元素配额）；第二轮 `*` 遍历补齐。附加面积过滤（> 视口 25% 踢掉）和上限保护（默认 600 个） |
| `dedupParentChild(results, domById)` | **父子去重** | 只看直接父级：父框盖住子框 80% 以上面积、或子是 `a/button` 而父是容器标签（div/span/li/p/section/article）时，去掉父留子。被去掉的元素连 unique_id 一起摘掉 |

**两轮收集的设计动机**：百度等页面中，导航栏、搜索框在文档顺序上排在前面。如果单轮按文档序遍历，导航栏的链接还没走完就可能吃光配额，正文的核心元素全部丢失。第一轮优先收录原生交互标签，确保它们不被导航栏挤掉。

---

### 模块 8：XPath 与文本提取（第 415–452 行）

| 函数 | 功能 | 说明 |
|------|------|------|
| `xpathFor(el)` | 计算元素的 XPath 路径 | 有 id 时直接返回 `//*[@id="..."]`；否则从元素向上逐层构建 `tag[n]` 路径 |
| `pseudoContent(el, pseudo)` | 提取伪元素文本 | 获取 `::before` 或 `::after` 的 `content` 属性值，截断到 40 字符 |
| `elementText(el)` | 元素的完整文本 | 优先级链：`innerText` → `value` → `aria-label` → `title` → `placeholder`，拼接 `::before` + `::after` 内容，压缩空白，截断到 160 字符 |

---

### 模块 9：四叉树与视觉分组（第 660–791 行）

截图上如果每个元素各画一个蓝框，视觉位置重叠的元素（如父子嵌套、图标+文字）会产生大量重叠框，LLM 无法辨认。这个模块把重叠的元素合并为同一个视觉目标。

| 组件 | 功能 | 说明 |
|------|------|------|
| `QuadTreeNode` | 四叉树数据结构 | 把空间递归四等分，支持高效的矩形范围查询。参数：maxElements=10、maxDepth=4 |
| `calculateBounds(items)` | 计算所有元素的包围盒 | 用于初始化四叉树的根节点范围 |
| `createRectangleForGroup(items)` | 计算一组的合并矩形 | 取所有成员矩形的最小包围矩形 |
| `findOverlapping(seed, quadTree, processed)` | 找 seed 的直接相交邻居 | **不做传递性扩展**（避免百度搜索结果链式合并成一个巨型分组）。只查四叉树中与 seed 矩形直接相交的元素 |
| `splitOversizedGroup(members)` | 超大分组兜底拆分 | 合并框盖住视口 40% 以上时，按位置排序后强制拆成子组（每组最多 `length/2` 个元素），保证每次拆分都严格变小 |
| `groupElementsVisually(items)` | **入口函数** | 建四叉树 → 逐元素找直接相交邻居 → 超大组拆分 → 返回分组列表 |

---

### 模块 10：画框与移除框（第 793–872 行）

这是最终呈现给 LLM 的视觉标注层。

| 函数 | 功能 | 说明 |
|------|------|------|
| `visibleAnnotationTargets()` | 收集当前视口内的标注目标 | 查询所有带 `unique_id` 属性的元素 → 过滤掉标注框自身、不可见元素、无可视矩形的元素 → 返回 `{id, rect}` 列表 |
| `removeBoundingBoxes()` | 移除所有标注框 | 删除 `msk-bounding-box-container` 容器。用 `removeChild` 而非 `Element.remove()` 以兼容 Prototype.js 风格的 polyfill |
| `drawBoundingBoxes(maxLabelIds)` | **核心渲染函数** | ① 移除旧框 → ② 收集标注目标 → ③ 视觉分组 → ④ 为每组创建蓝色 `<div>` 边框（`#2563eb`，2px）→ ⑤ 在框上方或框内左上角画标签显示 ID（如 `e5,e6+2` 表示组内有 4 个元素，显示前 2 个 + 剩余数）→ ⑥ 返回 `{groups, elements, labels}` |

**标注框样式**：蓝色边框 `#2563eb`、`z-index: 2147483000`（确保在最上层）、`pointer-events: none`（不影响页面交互）。标签背景同色、白色粗体 monospace 文字、高度 14px。

---

### 模块 11：滚动控制与页面清理（第 874–924 行）

Python 侧编排「滚一屏 → 画框 → 截图 → 移除框」循环所需的基础设施。

| 函数 | 功能 | 说明 |
|------|------|------|
| `safeWindowScroll(x, y)` | 安全滚动 | 兼容 `window.scroll` 和 `window.scrollTo`，使用 `behavior: "instant"` 避免动画 |
| `scrollInfo()` | 获取滚动状态 | 返回 `{scrollY, innerHeight, scrollHeight, hasMore, scrollable}`。`hasMore` 判断是否还能下滚（留 4px 容差）；`scrollable` 判断页面是否整体可滚动 |
| `scrollToTop()` | 滚回顶部 | 滚动到 (0,0) + 清空矩形缓存 |
| `scrollNextPage(overlap)` | 向下滚一屏 | 步进 = `innerHeight - overlap`，保留重叠像素避免跨屏元素在截图边界被切断 |
| `pageText(maxChars)` | 提取页面正文 | `document.body.innerText` 截断到 maxChars（默认 8000），供纯文本通道使用 |
| `removeAllUniqueIds()` | 清理页面 | 移除所有 `unique_id` 属性 + 重置计数器。任务结束时调用，恢复页面原始状态 |

---

### 模块 12：MutationObserver 增量抓取（第 926–1167 行）

用户点击/输入后，页面 DOM 会变化（如下拉菜单弹出、自动补全建议出现）。这个模块**只抓取因 DOM 变化而新增的可交互元素**，而不是每次全量扫描整个页面，节省 token 和延迟。

| 组件 | 功能 | 说明 |
|------|------|------|
| `getElementDomDepth(element)` | 计算 DOM 深度 | 从元素向上追溯到根节点，用于增量去重排序 |
| `isClassNameIncludesHidden(className)` | 检测隐藏态 class | 是否含 `hide` / `invisible` / `closed` |
| `isClassNameIncludesActivatedStatus(className)` | 检测激活态 class | 是否含 `open` / `active` |
| `_addIncrementalNodeToMap(parentNode, children)` | 增量节点处理 | 判断可交互性 → 分配 unique_id → 按深度归入 `_domDepthMap`。上限 3000 个节点防止烧穿额度 |
| `MutationObserver` 回调 | **DOM 变化监听** | 监听三类变化：① `attributes`（hidden/style/class 变化导致元素从隐藏变为可见）；② `childList`（新增子节点）；③ dropdown 相关元素（select/option/listbox）特殊处理——只要可见就视为新增 |
| `startIncrementalObserver()` | 启动监听 | 观察 `document.body` 的 `attributes` + `childList` + `subtree`，清空旧数据 |
| `stopIncrementalObserver()` | 停止监听 | 断开观察 + 清空残留记录 |
| `getIncrementElements()` | **获取增量结果** | 等待回调处理完毕（最多 2 秒）→ 按 DOM 深度从浅到深排序 → 同 id 去重（浅层优先）→ 验证元素仍在 DOM 上 → 刷新最新可见性状态 → 返回 `{new_elements, total_count}` |

---

## 三、window.__msk 暴露的 API 总表

注入完成后，以下方法挂载在 `window.__msk` 上供 Python 侧调用：

| API 方法 | 所属模块 | 功能简述 |
|---------|:-------:|---------|
| `collect(maxElements)` | 7 | 收集可交互元素，分配 unique_id，返回快照列表 |
| `drawBoundingBoxes(maxLabelIds)` | 10 | 画蓝色标注框和 ID 标签 |
| `removeBoundingBoxes()` | 10 | 移除所有标注框 |
| `visibleAnnotationTargets()` | 10 | 获取当前视口内已分配 unique_id 的可见元素 |
| `clearRectCache()` | 1 | 清空矩形缓存（滚动后调用） |
| `getVisibleClientRect(element, testChildren)` | 1 | 获取元素的可视区矩形 |
| `isInteractable(element)` | 6 | 判定元素是否可交互 |
| `scrollInfo()` | 11 | 获取当前滚动状态 |
| `scrollToTop()` | 11 | 滚回顶部 |
| `scrollNextPage(overlap)` | 11 | 向下滚一屏 |
| `pageText(maxChars)` | 11 | 提取页面纯文本 |
| `removeAllUniqueIds()` | 11 | 移除所有 unique_id，恢复页面原始状态 |
| `startIncrementalObserver()` | 12 | 启动 DOM 变化监听 |
| `stopIncrementalObserver()` | 12 | 停止 DOM 变化监听 |
| `getIncrementElements()` | 12 | 获取因 DOM 变化新增的可交互元素 |
| `isSelectableWidget(element)` | 5 | 判定元素是否为下拉/选择器控件 |
| `checkDisabledFromStyle(element)` | 4 | CSS 反推禁用状态 |
| `checkRequiredFromStyle(element)` | 4 | CSS 反推必填状态 |

---

## 四、数据流全景

```
页面全部 DOM 元素（可能上万个）
        │
        ▼
  ┌─ 模块 2: isVisible ─┐     排除不可见元素
  └──────────────────────┘
        │
        ▼
  ┌─ 模块 6: isInteractable ─┐     六层信号链过滤
  │   ├─ 模块 3: hover 推断    │     cursor:pointer 补充信号
  │   ├─ 模块 4: CSS 反推      │     disabled/required 补充信号
  │   └─ 模块 5: 控件识别      │     框架下拉框补充信号
  └────────────────────────────┘
        │
        ▼
  ┌─ 模块 7: collect ─┐
  │   两轮收集（核心标签优先）  │
  │   面积过滤（>25% 视口踢掉）│
  │   上限保护（≤600 个）      │
  └────────────────────────────┘
        │
        ▼
  ┌─ 模块 8: snapshot ─┐     分配 unique_id，采集属性/XPath/文本
  │   模块 1: getVisibleClientRect │     计算可视矩形
  └────────────────────────────────┘
        │
        ▼
  ┌─ 模块 7: dedupParentChild ─┐     父子去重（去包裹 div 留内部 <a>）
  └─────────────────────────────┘
        │
        ▼
  ┌─ 模块 10: drawBoundingBoxes ─┐
  │   模块 9: 四叉树视觉分组       │     重叠元素归为一组
  │   创建蓝色 <div> 边框 + ID 标签 │     Set-of-Mark 标注
  └────────────────────────────────┘
        │
        ▼
  Python 侧 page.screenshot() 截图
        │
        ▼
  LLM 看截图 + 编号 → 决策「点击 e37」
        │
        ▼
  ┌─ 模块 12: MutationObserver ─┐     捕获操作后新增的元素（下拉菜单等）
  └──────────────────────────────┘
        │
        ▼
  ┌─ 模块 11: scrollNextPage ─┐     滚一屏 → 重新画框 → 截图 → 循环
  └────────────────────────────┘
```
