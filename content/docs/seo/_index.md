+++
title = 'SEO 与站点发现'
linkTitle = 'SEO'
description = '语义化标记、结构化数据，以及让搜索引擎和 AI 爬虫找到内容的入口：站点地图、robots.txt、canonical、hreflang 与机器可读索引。'
weight = 60
+++

搜索引擎不读你的设计稿，它读模板输出的 HTML。这一章因此不谈"写作技巧"，只谈这个站点在模板层做了什么：页面用什么元素组织、结构化数据怎么生成、以及哪些文件是为了被机器发现而存在的。三页合起来，正好覆盖一次抓取从发现 URL 到理解正文的全过程。

## 这一节的三页

[语义化 HTML](/docs/seo/semantic-html/) 说的是最基础也最容易被忽略的一层：`header`、`nav`、`main`、`aside`、`footer` 这些地标怎么分工，一页为什么只有一个 `h1`，跳转链接和 `aria-current` 解决什么问题，以及日期为什么要写成 `<time datetime>` 而显示成中文。

[结构化数据](/docs/seo/structured-data/) 讲 `layouts/_partials/head/schema.html` 输出的那一个 `@graph`：Organization、WebSite、页面节点和 BreadcrumbList 各自负责什么，为什么把 `jsonify` 的结果直接塞进 `<script>` 会得到一个字符串字面量，以及怎么用 Google 和 schema.org 的工具验证。

[发现入口](/docs/seo/discovery/) 是清单式的：生成的 `robots.txt`、多语言的 `sitemap.xml`、每个页面的 Markdown 孪生、`llms.txt`、`pages.json`、`search.json`，加上 Google 与 Bing 的站长验证。

## 一次抓取会依次用到什么

把这一章的三页按抓取发生的顺序串起来，就能看出它们其实是同一条链上的不同环节：

1. 爬虫先读 `/robots.txt`。它由 `layouts/robots.txt` 生成，生产构建里才允许抓取，`Sitemap:` 那一行给出站点地图的绝对地址。
2. `/sitemap.xml` 在双语站点上是一份索引，指向每种语言自己的列表；列表里每个 URL 都带 `lastmod`，以及一组 `xhtml:link` 形式的 hreflang。
3. 真正取回一个页面时，`<head>` 里的 canonical 说明它是哪个地址的正本，hreflang 和 `x-default` 说明它有哪些语言版本。
4. 正文按地标读：`main` 是正文，`aside` 是补充信息，标题层级从唯一的 `h1` 往下不跳级。
5. 同一段 `<head>` 里的 JSON-LD 给出发布者、站点、页面类型和面包屑，机器不必再猜一次。
6. 不想解析 HTML 的客户端走另一条路：URL 后面接 `index.md` 拿 Markdown 孪生，或者直接读 `/pages.json`、`/search.json`、`/llms.txt`。

六步里只有第 4 步和正文写作有关，其余全在模板里。这就是为什么这一章通篇讲模板，而不是讲写作技巧。

## 为什么这是模板问题，不是内容问题

上面这些东西没有一样能靠写正文得到。canonical 必须是绝对地址，多语言互链要遍历翻译，面包屑要和可见的导航一致——它们都由 `layouts/_partials/head/` 下的几个模板负责，正文作者既不该也无法插手。

这也带来一个好处：修复一次，全站生效。给 `head/meta.html` 加一个字段，几百个页面同时获得它；反过来，模板里的一处错误也会同时出现在所有页面上，所以这一章强调"怎么验证"，而不是"记住这些字段"。

## 生产构建才有 index, follow

`head/meta.html` 里的 robots 指令取决于构建环境：默认是 `index, follow, max-snippet:-1, max-image-preview:large, max-video-preview:-1`，只要不是生产构建就整体变成 `noindex, nofollow`。这里的判断依据是 `hugo.IsProduction`，而**普通的 `hugo` 命令本身就等于生产环境**，只有 `hugo server` 或显式指定 `hugo -e development` 才是非生产。

{{< warning >}}
预览部署最常见的错误是"忘了加 noindex 头"，这里的做法把它反了过来：默认无头，生产才放开。如果你在 `hugo server` 里查看页面源码发现 `noindex`，那是设计如此，不是配置坏了——`layouts/robots.txt` 同样按环境切换，非生产环境直接 `Disallow: /`。
{{< /warning >}}

单个页面可以用 front matter 覆盖这两个判断：`robots` 直接写指令字符串，`noindex = true` 则强制退出索引。

## 怎么自检

一次性的检查比记住字段名更有用：

```bash
hugo --ignoreCache
cat public/robots.txt
grep -o 'rel="canonical"\|hreflang="[^"]*"\|application/ld+json' public/docs/assets/css/index.html
```

第一条确认 `robots.txt` 里 `Sitemap:` 是绝对地址；第二条确认任何一页的头部都有 canonical、成对的 hreflang 和一段 JSON-LD。再把 `public/sitemap.xml` 打开看一眼——多语言时它是一份 `<sitemapindex>`，每个语言各有自己的站点地图。三样都对，剩下的就是内容本身的事了。
