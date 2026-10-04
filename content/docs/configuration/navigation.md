+++
title = '导航'
linkTitle = '导航'
description = '菜单用 pageRef 还是 url、侧栏与面包屑从哪些配置读、分页怎么设，以及为什么菜单标签不进 i18n。'
date = 2026-01-14
weight = 20
difficulty = 'intermediate'
estimatedTime = 18
prerequisites = ['/docs/configuration/site-config/']
outcomes = ['分清 pageRef 与 url 的适用场景', '知道侧栏、面包屑、翻页各自读哪个配置', '把菜单标签放在正确的位置']
tags = ['Hugo', '导航']
+++

导航是页面上最先被看到、也最容易在本地化时出错的部分。它涉及四套彼此独立的机制：顶部菜单、文档侧栏、面包屑、上一页/下一页，再加上列表页的分页和底部的分类法。它们读不同的配置、用不同的模板，混在一起谈只会让人以为是一件事。下面逐套说。

## 菜单是数据，不是模板里的字符串

主菜单与页脚菜单都定义在 `config/_default/` 下的语言菜单文件里：`menus.zh-cn.toml`、`menus.en.toml`。每个条目至少要有 `name` 和 `weight`，再加上一个指向目标的方式。渲染由 `layouts/_partials/menu.html` 完成，它按 `weight` 排序并把子项递归展开，主菜单和页脚菜单共用同一个模板，只是传入的菜单 ID 不同。

条目里最有价值的一行是 `weight`：菜单顺序完全由它决定，文件里的书写顺序不参与排序。改顺序就是改数字，不需要移动文本块。

## pageRef 与 url

指向站内页面时用 `pageRef`，指向站外地址时用 `url`。站点的主菜单里两种写法都在：

```toml
[[main]]
  name = '文档'
  pageRef = '/docs'
  weight = 10

[[main]]
  name = 'Hugo 官网'
  url = 'https://gohugo.io/'
  weight = 90
```

区别不只是"内部还是外部"，而是这一项能不能参与当前页判定。高亮当前菜单项需要回答两个不同的问题，`layouts/_partials/menu.html` 两个都问：

- `IsMenuCurrent` ——当前页**就是**这一项。命中时模板给链接加上 `is-current` 类和 `aria-current="page"`，屏幕阅读器会把这一项读成"当前页"。
- `HasMenuCurrent` ——当前页**位于**这一项**之下**。命中时加上 `is-ancestor` 类，于是你读 `/docs/configuration/navigation/` 时，顶栏的"文档"仍然亮着。

两个判断都要求菜单项关联到一个页面对象，也就是要求 `pageRef`。顺带得到第二个好处：页面 URL 变化时菜单自动跟上——把 `/docs/` 改成 `/documentation/`，菜单不需要改。用 `url` 写的站外条目没有页面可比，模板会按外链处理并跳过这两个判断；这也是为什么菜单模板先判断 `strings.HasPrefix .URL "http"`，再判断当前项，顺序反了会把外链也当成内部链接来比较。

{{< warning >}}
`pageRef` 写的是内容路径，不是最终 URL，但它必须对应一个真实存在的页面。写错时 Hugo 不会阻止构建，只是在渲染菜单时拿不到页面对象，于是当前项高亮和层级判定同时失效——症状是"菜单能用，但永远不高亮当前页"。
{{< /warning >}}

## 菜单标签为什么不进 i18n

界面文案走 `i18n/` 目录（例如"上一篇""下一篇""目录"），但菜单标签不走。原因有两条，第二条更重要：

- 菜单标签是导航，不是模板内部字符串。把它塞进翻译目录意味着每次调整导航都要同时改配置和翻译文件；
- 缺一个翻译键，Hugo 会在**每个页面**上输出一条 `MISSING_TRANSLATION` 警告。菜单项本来就少，为它们承担"每次构建刷一屏警告"的风险并不划算。

更实际的是，英文站点通常不是逐条翻译菜单，而是替换或重排条目——两种语言的菜单结构本来就可以不一样。分成两个语言文件之后，`menus.en.toml` 可以有自己的顺序和条目，而不必迁就中文版。

## 侧栏、面包屑与翻页

这三者的开关都在 `config/_default/params.toml` 的 `[params]` 里，行为在模板里：

- **侧栏**由 `[params.nav] sidebarSections = ['docs']` 决定哪些顶级章节显示章节内导航，因此只有文档区有侧栏，博客区没有。`layouts/_partials/sidebar.html` 取当前页所属章节的根节点，然后按 `weight` 遍历 `.Pages.ByWeight`——章节和普通页面在同一个列表里排序，所以"章节首页、若干页、子章节"这种交错顺序是稳定的。侧栏只展开当前页所在的那条路径，其余分支折叠。
- **面包屑**由 `[params.ui] showBreadcrumbs` 控制，模板沿 `.Ancestors.Reverse` 从首页走到当前页。同一个导航轨迹还会以 `BreadcrumbList` 结构化数据的形式写进页面头部，两者必须一致——可见面包屑和结构化数据打架属于错误，不是排版问题。
- **上一页 / 下一页**由 `[params.ui] showPrevNext` 控制，用 `.PrevInSection` 和 `.NextInSection`。它们遵循所在章节自己的排序（先 `weight`，再 `date`），和侧栏的顺序同源，所以翻页和侧栏永远不会互相矛盾。

## 分页

列表页的分页大小由 `config/_default/hugo.toml` 里的 `[pagination] pagerSize = 2` 设定。这个值刻意取小，为的是让分页控件在演示站点上真的出现——`/blog/` 只有两篇文章，所以首页就翻页了；真实站点应该取 10 以上。

模板侧的分工值得记一笔：`layouts/section.html` 用 `.Paginator` 而不是 `.Paginate`。原因是在 `head/meta.html` 里已经取过一次 `.Paginator`，用来生成带页码的 `<title>` 和 canonical 地址；同一个页面再调用 `.Paginate` 并传入另一个集合，会得到两个互相冲突的分页器。渲染本身交给 `layouts/_partials/pagination.html`，它在页数超过八页时只画出首页、末页和当前页附近的窗口，中间用省略号表示断档，避免六十页的归档画六十个链接。

## 分类法

`config/_default/hugo.toml` 的 `[taxonomies]` 声明了三个：`tag = 'tags'`、`category = 'categories'`、`series = 'series'`。Hugo 会为每个分类法生成列表页和词条页，即使没有任何内容使用它们。文章通过 front matter 关联：

```toml
tags = ['Hugo', '主题']
categories = ['工程实践']
series = ['从零搭一个 Hugo 站点']
```

`[params.ui] showTags` 控制文章页是否显示标签。分类法页面本身由 `layouts/taxonomy.html` 和 `layouts/term.html` 渲染，词条列表交给 `layouts/_partials/terms.html`。如果你打算完全不用分类法，可以在配置里加上 `disableKinds = ['taxonomy', 'term']`，让 Hugo 不再生成这些页面；但在此之前请先确认没有模板或菜单引用它们。

导航相关的配置就这些。改完之后跑一次构建，警告里通常能立刻看出菜单项指向了不存在的页面，或者某个章节的 `weight` 与邻居重复了。
