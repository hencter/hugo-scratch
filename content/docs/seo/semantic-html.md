+++
title = '语义化 HTML'
linkTitle = '语义化 HTML'
description = '地标元素、唯一的 h1、跳转链接、aria-current、可读的日期与语言属性，以及图片、表格、打印和减少动效的处理。'
date = 2026-02-25
weight = 10
difficulty = 'beginner'
estimatedTime = 12
prerequisites = ['/docs/', '/docs/seo/']
outcomes = ['说出每个地标元素在这份布局里的职责', '知道标题层级和锚点是怎么生成的', '能解释日期为什么要写成 time datetime']
tags = ['SEO']
+++

写 Markdown 的人通常感觉不到这一层：正文里只写 `##`，页面却自动有了地标、跳转链接、当前项标记和机器可读的日期。这些全部来自模板。这一页逐个说明它们，因为每一处都同时服务两个读者——屏幕阅读器和爬虫——而且都只需要在模板里做对一次。

## 地标：谁包住谁

`layouts/baseof.html` 是这份分工的合同。文档开头是 `<html lang="…" dir="…">`，紧接着一个跳转链接，然后是头部、内容网格和页脚。每个页面模板只负责填 `main` 块，位置由这里决定：

- `layouts/_partials/header.html` 输出真实的 `<header class="site-header">`，里面是 `<nav id="site-nav" aria-label="主导航">`。移动端的展开按钮只切换可见性，菜单本身始终在 HTML 里。
- `layouts/_partials/sidebar.html` 与 `layouts/_partials/toc.html` 各输出一个 `<nav>`，用 `aria-label` 和 `aria-labelledby` 互相区分：前者是"本节导航"，后者是"本页目录"。同一个页面上有三个导航地标是很正常的，前提是它们各自有名字。
- `layouts/_partials/facts.html` 输出 `<aside class="facts">`。难度、预计时间、前置知识和收获属于正文的补充信息，所以用 `aside`，而不是塞进标题下面的一段文字。
- `layouts/_partials/footer.html` 输出 `<footer class="site-footer">`，里面对机器可读文件的链接同时服务读者和爬虫。

{{< note >}}
地标的价值在于"跳过"。读屏用户不需要在每一页从头听一遍导航，他可以直接跳到 `main`；爬虫也依赖这些边界判断哪一段是正文。所以 `<div class="header">` 和一个真正的 `<header>` 在视觉上完全一样，在语义上没有替代关系。
{{< /note >}}

## 一页一个 h1，标题不跳级

`h1` 由页面模板给出，`layouts/page.html` 和 `layouts/section.html` 都把它写在 `page__header` 里，内容用 `.Title`。所以正文的 Markdown 必须从 `##` 开始——再写一个 `#` 就会出现两个一级标题，标题大纲也随之断掉。

正文里的标题由 `layouts/_markup/render-heading.html` 渲染。它做三件事：保留 Hugo 算好的 `.Anchor` 作为 `id`，但允许 `{#custom-id}` 覆盖它；把 `{.class}` 之类的块属性透传到元素上；在标题末尾追加一个指向自身的锚点链接，带 `aria-label`，所以它是一个有可访问名字的真链接，而不是装饰性的 `#` 字符。

目录的深度由 `config/_default/hugo.toml` 里的 `[markup.tableOfContents]` 决定，当前是 `startLevel = 2`、`endLevel = 3`，正好对应正文能用的两级标题；`layouts/_partials/layout/flags.html` 还会数一数标题数量，不足三条就不渲染侧边目录。

## 跳转链接与当前项

跳转链接是键盘用户拿到页面之后的第一站。它写在 `baseof.html` 的第一行，指向 `#main`，而 `<main id="main" … tabindex="-1">` 上的 `tabindex="-1"` 是配套的：没有它，焦点无法通过链接落到 `main` 上。`.skip-link` 在 `assets/css/base.css` 里默认移出视口，获得焦点时回到左上角。

"当前项"用 `aria-current` 表达，本站有三种用法，都对应真实含义：

- `layouts/_partials/menu.html` 里，当前页面对应的菜单项标 `aria-current="page"`；只是当前页位于其下的祖先项标 `aria-current="true"`——两者含义不同，所以值也不同。
- `layouts/_partials/sidebar.html` 给当前页面的侧栏链接标 `aria-current="page"`，同时只展开当前页所在的那条分支。
- `layouts/_partials/breadcrumbs.html` 给最后一个面包屑（一个不可点的 `<span>`）标 `aria-current="page"`。
- `assets/js/modules/toc.js` 在滚动时给读者正在看的那一节标 `aria-current="true"`，离开就移除。

## 机器可读的日期与语言

`layouts/_partials/page-meta.html` 里每个日期都写成 `<time datetime="YYYY-MM-DD">`，显示文本则来自 `site.Params.dateFormat`——中文是 `2006 年 1 月 2 日`。两者故意不同：显示要符合读者习惯，`datetime` 要能被机器解析，所以改显示格式永远不会影响数据。

这一层有两个容易踩的坑，模板里都已经处理：`.Date` 是结构体而不是指针，`{{ with .Date }}` 永远为真，所以每个日期都用 `.IsZero` 判断，未填日期的页面不会打印出 `0001-01-01`；"更新于"只在 `Lastmod` 与 `Date` 不是同一天时才出现，避免同一行重复两个一样的日期。

语言相关的属性写在 `baseof.html` 的 `<html>` 上：`lang` 取 `site.Language.Locale`（中文页是 `zh-CN`，英文页是 `en-US`），`dir` 取 `site.Language.Direction` 并在缺失时回落到 `ltr`。这两个值同时被 hreflang、Open Graph 的 `og:locale` 和 JSON-LD 复用，所以它们只有一处出处。

## 图片、表格与辅助文本

正文里的图片经 `layouts/_markup/render-image.html` 输出。`alt` 取 Markdown 方括号里的文本并去掉标记，所以写 `![站点 CSS 目录结构](…/tree.png)` 时，读屏用户听到的就是那句话。Hugo 知道原始尺寸时会补上 `width` 和 `height`，这就是图片加载时布局不跳的原因；此外统一带 `loading="lazy"` 和 `decoding="async"`。给图片写标题（`![alt](img.png "说明文字")`）会让它变成一个带 `<figcaption>` 的 `figure`，正文里能看见的那行说明和替代文本因此是两回事。

表格经 `layouts/_markup/render-table.html` 输出，外面永远包一层 `.table-wrap`，窄屏上横向滚动的是表格而不是整个页面。块属性会落到 `<table>` 上，例如 `{#size-table .table--compact}`，这样你就能用 `#size-table` 从别处引用它。

{{< tip >}}
Markdown 语法里没有表格标题元素，这一点不需要绕过去：在表格前面用一句话或一个小标题说明它是什么，再给它一个 `id`，读者和爬虫都能定位。比硬塞一个加粗的行当标题要清楚得多。
{{< /tip >}}

需要"只给辅助技术看"的文本时，用 `assets/css/base.css` 里的 `.visually-hidden` 或 `.sr-only`。它们把内容移出视口而不是 `display: none`，所以读屏软件仍然读得到；选择器上带 `:not(:focus, :active)`，元素一旦被聚焦就会显示出来——这正是跳转链接这类控件的实现方式。

## 打印样式与减少动效

`assets/css/print.css` 在 `@media print` 里隐藏页头、页脚、侧栏、目录、搜索框、翻页和复制按钮，把三栏布局压成一栏，并把外链地址用 `::after` 附在链接后面——纸上点不了链接，目标地址就得写出来。需要只在纸上出现的元素，可以用 `.print-only`。

动效尊重系统设置：`base.css` 顶部的 `@media (prefers-reduced-motion: reduce)` 把 `scroll-behavior` 改回 `auto`，并把动画与过渡时长压到 `0.01ms`。这条规则对所有元素生效，所以新加的交互动效不需要各自处理一遍。
