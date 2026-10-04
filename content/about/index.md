+++
title = '关于'
linkTitle = '关于'
description = '这个仓库是什么、给谁用、刻意不做什么，以及许可与地址。'
date = 2026-01-05
weight = 10
+++

## 这个仓库是什么

Hugo Scratch 是一个把 Hugo 常用特性**真的跑起来**的站点，而不是一份特性清单。
每个特性都对应仓库里真实存在的文件：语义化的 HTML 结构、完整的 SEO 头、
中英双语导航与内容、按需加载的客户端搜索、可切换的明暗主题、
一组短代码与 Markdown 渲染钩子，以及完全由 Hugo 构建的 CSS/JS 管线。

它同时也是一份文档：正文里出现的每一个文件名、配置键和模板路径都能在仓库里找到，
所以「照着改一遍」和「读一遍」是同一件事。

站点用 `hugo new site` 与 `hugo new theme` 起头，主题在
<https://github.com/hencter/hugo-scratch-theme>，作为子模块挂在
`themes/hugo-scratch-theme`；内容与配置属于
<https://github.com/hencter/hugo-scratch>。线上地址是
<https://scratch.hugozh.cn/>。当前版本是 `v1.11.0`。

## 给谁用

三类读者：

1. **正在搭第一个 Hugo 站点的人。** [快速开始](/docs/start/)给出从克隆到看到页面的完整路径，
   [目录结构](/docs/start/directory-structure/)解释每个目录为什么在那里。
2. **想抄一段配置的人。** 从[站点配置](/docs/configuration/site-config/)一节的
   `config/_default/hugo.toml` 开始，[Assets 与 Hugo Pipes](/docs/assets/css/)
   给出两条构建管线的完整写法。
3. **读文档的机器。** 每个页面除 HTML 之外还输出 Markdown 孪生页和 `pages.json`，
   页面的难度、预计用时、前置条件、学习产出都写在 front matter 里而不是正文里，
   所以一个代理不需要解析 HTML 就能判断某一页该读还是该查。

## 刻意不做的三件事

**不加载第三方脚本。** 页面只请求同源的资源：样式表、脚本、
`search.json` 与页面本身的图片。没有 CDN、没有字体服务、
没有评论插件。YouTube 短代码渲染的是一个**链接**而不是播放器嵌入，
所以到读者点出去之前，一个第三方请求都不会发生。

**默认不做统计。** `layouts/_partials/analytics.html` 只在
`[params.analytics]` 填了值、**并且**是生产构建时才输出标签。
仓库里的两个值都是空字符串，所以现在这个站点的访问不会被任何分析服务记录。
这在我的看法里不是省事，而是默认值该有的样子：要收集什么，得有人明确写下来。

**只多一个 Node 依赖，而且只在安装时用一次。** `package.json` 里只有 `tailwindcss` 与 `@tailwindcss/cli` 两个包，用 `npm ci` 按锁文件装一次；之后的每次构建都不再需要网络。设计系统由 Hugo 内置的 `css.Build` 内联，脚本由内置的 `js.Build`（esbuild）打包，所以仓库里没有 SCSS、没有 PostCSS 插件、也没有任何构建脚本配置文件——`css.TailwindCSS` 是 Hugo 的官方集成，不是我们自己搭的管线。

{{< tip >}}
想把这份脚手架当作模板用，最省事的路径是 `hugo new content` 配合主题自带的
archetype：`hugo new content blog/my-post/index.md --kind blog` 会直接生成
带 `date`、`authors`、`tags`、`categories`、`series` 的 front matter。
{{< /tip >}}

## 许可

站点的**正文**（`content/` 下的文字与图片）采用
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)：
转载、改写、商用都可以，署名即可。

**主题与代码**——主题仓库、`layouts/`、`assets/` 与正文里的代码片段——采用 MIT 许可，
可以拿去改、拿去发，不需要保留什么仪式感，只保留版权声明。

具体的条款写在[许可](/legal/license/)一页；
站点在浏览器里会请求什么、不会请求什么，写在[隐私](/legal/privacy/)一页。
两页都随实现一起维护：改了模板或配置，就同时改这两页。
