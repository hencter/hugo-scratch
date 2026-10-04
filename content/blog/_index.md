+++
title = '博客'
linkTitle = '博客'
description = '这个站点的构建笔记：脚手架、Hugo Pipes 与双语站点，每篇只讲一个踩过的坑。'
weight = 20
groupByYear = true
+++

## 这里的笔记是怎么排的

这一节放的是这个站点自己的构建笔记：每篇文章对应一个**具体问题**，而不是一份特性清单。
一篇讲完一个主题，代码块里出现过的文件名都能在仓库里找到同名文件。

`content/blog/_index.md` 的 front matter 里写了 `groupByYear = true`，
`layouts/_partials/page-list.html` 读到这个值后改用
`collections.GroupByPublishDate` 把文章按年份分组——所以下面的列表是年份小节，
而不是一条平铺的清单。去掉那一行，分组标题会一起消失。

{{< note >}}
所有文章都按同一套 front matter 写：`date`、`authors`、`tags`、`categories`、`series`。
分类法在 `config/_default/hugo.toml` 的 `[taxonomies]` 里声明，`series` 是其中自定义的一项，
它把这一系列文章串成一条阅读顺序。
{{< /note >}}

## 按年份分组意味着什么

写文章时只需要填对 `date`，年份标题是渲染时算出来的：年份来自
`.Date.Format "2006"`，不是写在正文里的字符串。这也意味着**日期不可以随手改**：
把一篇文章的 `date` 改成另一年，它会安静地跳到另一组去，而没有任何地方会报错。

分组标题本身是一个 `##` 级别的标题，模板给它钉了 `id="year-2026"` 这样的锚点，
所以从外部链接到「2026 年那批文章」是可行的。

## 现在这一系列写了什么

目前是一个四篇的系列「从零搭一个 Hugo 站点」：

- [为什么从脚手架开始](/blog/hello/)：`hugo new site` 与 `hugo new theme` 究竟生成了哪些文件，以及主题为什么被拆成独立仓库。
- 用 Hugo Pipes 拼出 CSS 与 JS：`css.Build` 内联主题的设计系统、`css.TailwindCSS` 跑 Tailwind v4 的 CLI、`js.Build` 用 esbuild 打包，以及为什么只有 Tailwind 那一段需要 `npm ci`。
- 一个站点的两种语言：`.en.md` 配对规则、语言切换器的回退、`hreflang` 与 `og:locale` 的差异。
- 剩下的几篇随站点功能补齐，写一篇发一篇。

## 这里假定的读者

读者假定你会写 Markdown，也假定你装好了 Hugo——本站要求
0.146.0 或更高版本[^1]，当前的构建版本是 {{< version >}}。
不假定你会写 Go 模板：凡是需要改模板的地方，文章都会指出文件名和改动位置。

## 怎样把这些笔记当文档用

每一页除了 HTML 还输出一份 Markdown 孪生页，页面底部的「Markdown 源文」链接直接指向它，
所以一篇文章可以整段粘进编辑器或另一个仓库，不必从渲染后的页面里刮文本。

> [!TIP]
> 首页的搜索框索引的是全站文本；搜「Pipes」「`@params`」这类具体标识符，
> 比搜「构建」更快落到某一篇。

{{< details title="为什么不做评论区" >}}
评论需要一个第三方服务、一段第三方脚本，以及一份随之而来的隐私说明。
现在 `layouts/_partials/comments.html` 只渲染一个指向仓库 issue 的链接：
想讨论的读者点过去，不想讨论的读者不会因此多下载一个字节。
{{< /details >}}

[^1]: [Hugo 安装说明](https://gohugo.io/installation/)
