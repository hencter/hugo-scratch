+++
title = 'JavaScript 管线'
linkTitle = 'JavaScript'
description = 'main.js 怎样被 js.Build 打成单个 IIFE、@params 如何把搜索索引地址和界面译文注进代码、七个模块各自负责什么。'
date = 2026-02-25
weight = 20
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/', '/docs/assets/']
outcomes = ['说清 @params 送了什么进浏览器', '知道七个模块分别负责哪一件事', '能新增一个模块并接到入口上']
tags = ['JavaScript']
+++

这条管线看起来比 CSS 那条复杂，其实只多做了一件事：它把站点的配置和译文一起打包进脚本。理解这一点之后，`@params`、双语两个产物、以及为什么搜索框不需要任何数据属性，就都变成了同一个答案。

## 一个入口，esbuild 打成一个文件

`layouts/_partials/head/js.html` 从头到尾只读一个文件——`assets/js/main.js`，然后交给 `js.Build`。esbuild 在 Hugo 内部跑，把 `import` 出来的整张模块图打成一个文件，所以浏览器只有一次脚本请求，也不需要模块加载器。构建选项分开发和生产两种取值：

{{% tabs %}}
{{% tab "hugo server（开发）" %}}
```text
minify=false   target=es2020   format=iife
sourceMap="linked"   params={…}
```
产物保留链接式 source map，浏览器控制台里的报错指回原始模块，而不是打包后的某一行。脚本以普通 `<script src="…" defer>` 引入，不加指纹。
{{% /tab %}}
{{% tab "hugo（生产）" %}}
```text
minify=true    target=es2020   format=iife
sourceMap="none"     params={…}
```
压缩之后再做 `fingerprint "sha384"`，`integrity` 与 `crossorigin="anonymous"` 一起写进 `<script>` 标签，`defer` 两种情况都有，所以脚本永远不会阻塞解析。
{{% /tab %}}
{{% /tabs %}}

`target` 是 `es2020`，`format` 是 `iife`：产物是一个立即执行的函数，不往全局挂任何东西，也不会与页面里可能存在的其他脚本抢全局名字。需要确认产物是否生成，看 `public/js/main.<hash>.js` 就够了。

## @params：把配置和译文注进代码

`js.Build` 的 `params` 选项会被 Hugo 变成一个虚拟模块，名字固定叫 `@params`，入口文件第一行就把它整个取进来：

```js
import * as params from '@params';
```

模板里塞进这个字典的只有两样东西。第一样是搜索索引的地址，取自首页 `search` 输出格式的 `RelPermalink`，用 `with` 包着——站点若把 `search` 从 `[outputs]` 里删掉，这里会安静地变成空字符串，而不是让构建失败。第二样是 `i18n` 子字典，六条界面文案（复制、已复制、搜索加载中、无结果、结果数、搜索提示）在构建时通过 `T` 取好译文，直接写进代码。

{{< note >}}
`params` 是**构建期**注入的，不是运行时的配置接口。改 `i18n/` 里的译文或 `params.toml` 之后必须重新构建，脚本才会带上新值；反过来，因为译文在打包那一刻就固定了，浏览器不需要为了几条界面文案再发一次请求。
{{< /note >}}

## 七个模块各自负责什么

`assets/js/main.js` 只做三件事：导入模块、导入 `@params`、在 DOM 就绪后依次调用每个模块的 `init`。真正的工作都在 `assets/js/modules/` 下：

`theme.js`
: 明暗与跟随系统的三态切换，写入 `data-theme` 和 `light`/`dark` 类。初始值不归它管——`head/theme-init.html` 的内联脚本必须在首次绘制前跑完，而打包后的模块按定义是延迟的。

`nav.js`
: 只切换移动端导航的可见性。头部的 `<nav>` 和菜单列表是服务端渲染的真实标记，所以关掉 JavaScript 之后导航依然可用、也依然能被爬虫读到。

`search.js`
: 在读者第一次打开搜索框时才去取 `search.json`，匹配用的是大小写折叠后的子串比较——中日韩文本没有词边界，分词在这里没有意义。

`toc.js`
: 目录的滚动高亮，用 `IntersectionObserver` 而不是 `scroll` 监听，读者滚动时主线程没有额外工作。

`copy.js`
: 代码块的复制按钮。按钮由代码块渲染钩子在服务端输出，所以脚本没加载时它只是不响应，不会凭空消失。

`tabs.js`
: 给标签页补上键盘导航，并隐藏未选中的面板。所有面板本来就在 HTML 里，爬虫看到的是完整内容。

`back-to-top.js`
: 滚过大约一屏之后把回顶按钮显出来，只改一个数据属性，样式全部留在 CSS 里。

## 为什么每种语言各有一个包

`params` 里带着译文，而每种语言的译文不同，所以 `js.Build` 的结果也必然不同：一次双语构建会在 `public/js/` 下留下两个哈希不同的文件，中文页面引用其中一个，英文页面引用另一个。

这不是浪费。若把译文从脚本里挪走，就得在 HTML 上给每个按钮挂 `data-label-*` 属性，或者让页面再发一次请求去取一份语言包——前者把文案摊进标记，后者把首屏多一次往返。构建期注入把代价一次性付在构建上，读者拿到的是一个自包含的文件。指纹机制让两个产物各自缓存，互不干扰；改中文译文只会让中文那份的文件名变化。

## 加一个模块

过程是固定的，改两个地方：

{{% steps %}}
{{% step "新建模块文件" %}}
在 `assets/js/modules/` 下建文件，导出唯一的初始化函数，例如 `export function initFoo() { … }`。入口之外的模块不要自己监听全局事件，也不要假设 DOM 已经就绪——那是入口的职责。
{{% /step %}}
{{% step "接进入口" %}}
在 `assets/js/main.js` 里加一行 `import { initFoo } from './modules/foo.js';`，然后在 `ready(() => { … })` 的回调里调用 `initFoo()`。`ready` 已经处理了脚本晚于 DOM 解析的情况。
{{% /step %}}
{{% step "重建并确认" %}}
`hugo --ignoreCache` 之后看 `public/js/`：文件名哈希变了，说明新模块进了包。如果脚本报错说找不到元素，先确认模板里那个挂钩用的数据属性真的渲染出来了。
{{% /step %}}
{{% /steps %}}

## 这套管线不做的事

两件事值得明说，因为很容易从别的站点带过来。第一，脚本这一段落没有延迟渲染：`layouts/_partials/head.html` 里只有样式表被包在 `templates.Defer` 里（Tailwind 要等 `hugo_stats.json` 写完才敢编译），`head/js.html` 是直接调用的，它在解析 `<head>` 时就把标签产出，`defer` 只负责执行的时机。第二，没有前端框架，也没有水合过程——每个模块都是拿到节点、加监听、改属性就结束，页面在脚本加载之前已经是一份可读、可点的文档。想加一个需要状态树的交互组件，先想清楚它值不值得破坏这个前提。
