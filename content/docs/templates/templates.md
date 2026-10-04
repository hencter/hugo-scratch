+++
title = '模板与查找顺序'
linkTitle = '模板与查找顺序'
description = 'baseof 定义契约、partial 返回值、上下文的 $ 与 .、以及 and/or 会求值每个参数这件事。'
date = 2026-02-20
weight = 10
difficulty = 'advanced'
estimatedTime = 25
prerequisites = ['/docs/templates/', '/docs/start/directory-structure/']
outcomes = ['说清一个页面从内容到 HTML 经过哪些模板', '写一个返回值而不是打印标记的 partial', '避开 and/or、default 与 nil 的三个坑']
tags = ['Hugo']
+++

这个主题没有 `layouts/_default/`。它用的是当前模板系统：`baseof.html` 定义文档骨架，页面类型模板只提供 `main` 块，partial 放在 `layouts/_partials/`，短代码放在 `layouts/_shortcodes/`，渲染钩子放在 `layouts/_markup/`。

## 查找顺序：先具体，后通用

一次渲染要回答两个问题：**哪个输出格式**、**哪个模板**。找 `page.html` 时，Hugo 从最具体的位置往外退：

1. `layouts/docs/content/page.html`——类型 + 章节
2. `layouts/docs/page.html`——类型
3. `layouts/docs/content/single.html`——类型 + 章节
4. `layouts/docs/single.html`——类型
5. `layouts/page.html`
6. `layouts/single.html`

`layouts/docs/page.html` 优先于 `layouts/page.html`，这一点在这个沙箱里实测过：往 `layouts/docs/` 放一个只写 `page.html` 的文件，`/docs/content/front-matter/` 立刻改用它，而 `/docs/content/` 这个章节页不受影响——章节页找的是 `section.html`，不参与这条链。

{{< note >}}
本主题的文件名与旧写法一一对应：`single.html` 对应 `page.html`，`list.html` 对应 `section.html` / `taxonomy.html` / `term.html`，`index.html` 对应 `home.html`。两套名字都还能用，但一个项目里只该留一套——混用之后，改了一个文件却看不到任何变化。
{{< /note >}}

## `baseof.html` 是契约，页面只填 `main`

`layouts/baseof.html` 拥有 `<html>`、`<body>`、页头、三栏栅格与页脚，并在中间留了一个空位：

```go-html-template
<main id="main" class="page {{ if .IsPage }}page--reading{{ else }}page--wide{{ end }}" tabindex="-1">
  {{ block "main" . }}{{ end }}
</main>
```

`layouts/page.html`、`layouts/section.html`、`layouts/home.html` 等各自只写 `{{ define "main" }}…{{ end }}`。这个契约还留了两个给站点用的块：`head-extra`（在 `layouts/_partials/head.html`）和 `scripts`（在 `baseof.html` 底部），站点可以在自己的模板里定义它们来追加标签或脚本，而不必覆盖整个 `baseof.html`：

```go-html-template
{{ define "head-extra" }}<link rel="me" href="https://example.com/@me">{{ end }}
```

`page.html` 的 `main` 块本身很短：面包屑、标题、`banner`、`page-meta`、`facts`、`.Content`，然后是分类法、上一页/下一页与评论。

## 一次渲染的完整路径

以本页 `/docs/templates/templates/` 为例：

1. `baseof.html` 调 `partial "layout/flags.html" .`，拿到 `{sidebar: true, toc: true}`，据此加上 `layout--with-sidebar layout--with-toc` 两个类；
2. 同一次渲染里，`partial "head.html" .` 依次调 `head/meta.html`（标题、描述、canonical）、`head/alternates.html`（RSS 与 Markdown 孪生页的 `<link rel="alternate">`、hreflang）、`head/opengraph.html`、`head/schema.html`；
3. `page.html` 定义 `main`，渲染面包屑与信息面板；
4. `.Content` 触发 Markdown 渲染，每个标题、代码块、表格、链接与图片分别经过 `layouts/_markup/` 下的钩子；
5. `partial "toc.html" .` 读取 `.Fragments.Headings` 重建目录——注意不是 `.TableOfContents`，那个只返回一整段现成的 HTML 字符串；
6. `partial "scripts.html" .` 之后落到 `{{ block "scripts" . }}` 的空位。

{{< tip >}}
`.Content` 是**惰性**的：模板里不写它，Markdown 就不会渲染，渲染钩子也不会跑。调试钩子时先确认页面上真的调了 `.Content`。
{{< /tip >}}

## partial：打印，或者返回值

默认的 partial 把结果直接写进输出流，例如 `partial "icon.html" "search"`。也可以用 `return` 交回一个值，调用方拿变量接住：

```go-html-template
{{ $flags := partial "layout/flags.html" . }}
{{ if $flags.toc }}…{{ end }}
```

`layout/flags.html` 返回 `{sidebar, toc}`，`resolve-image.html` 返回 `{url, width, height, resource}`。返回值的好处是把判断集中在一个文件里：`baseof.html`、`sidebar.html`、`toc.html` 都问同一份答案，不会各自算一遍还算出不同结果。还有一种不落盘的 partial：**行内 partial**，把 `{{ define "_partials/inline/…" }}` 写在用它的文件里，供同一文件递归调用。`layouts/_partials/sidebar.html` 与 `toc.html` 都用这个办法写递归树——名字以 `_partials/` 开头才算行内 partial，其它名字会被当成普通模板。

## 上下文、字典与切片

- `.` 是当前上下文：在页面模板里是页面对象，在 `range` 里是当前元素；
- `$` 是模板最开始的那个上下文，永远指向页面。嵌套 `range` 里写 `$.Page` 最稳；
- `dict` 造字典，`slice` 造切片，两者是 partial 传多参数的常规做法：`partial "card.html" (dict "page" .)`；partial 只接受一个上下文参数，所以多个值一律打包成 `dict`，`layouts/_partials/page-list.html` 就是 `(dict "pages" $pages "group" $group)`。

## 守卫，以及三个真实的坑

```go-html-template
{{ with site.Params.tagline }}<p>{{ . }}</p>{{ end }}
{{ $show := index $ui "showSidebar" }}
{{ if eq $show nil }}{{ $show = true }}{{ end }}
```

{{< danger "坑一：and / or 会求值每个参数" >}}
`and $p $p.Title` 在 `$p` 为 nil 时照样 panic——`$p.Title` 已经被求值了，`and` 只是收下两个结果。要短路就用嵌套 `with`，或先算出一个安全的字符串：

```go-html-template
{{ $title := "" }}
{{ with $p }}{{ $title = .Title }}{{ end }}
{{ if and $p $title }}…{{ end }}
```
{{< /danger >}}

{{< warning "坑二：default true 表达不了显式的 false" >}}
`default` 只在值为空时接手。调用方写下的 `false` 和「没写」在它眼里一样，于是被替换成 `true`。开关要看键是否存在，就用 `isset` 或 `index` 加 `eq ... nil`——`flags.html` 对 `showSidebar` 正是这么做的。
{{< /warning >}}

第三个坑是日期：`.Date` 是 `time.Time` 结构体，用 `with` 判断它永远为真，所以判断「有没有日期」必须写 `.Date.IsZero`。`page-meta.html` 里每个日期都带这个判断。

## `.Store`：在短代码之间传递状态

`.Store` 是挂在页面或短代码上的临时存储，写一个键、读一个键，生命周期只到这次构建结束。`layouts/_shortcodes/tabs.html` 用它解决了一个真实问题：每个 `tab` 子短代码把 `{id, title}` 追加到 `$parent.Store` 的 `tabTitles` 键上，而嵌套短代码是**由内向外**渲染的，所以等父短代码执行时，所有子项的标题都已经写进去了；父短代码读到完整的 `tabTitles`，就能把标签按钮在服务端一次性渲染出来——爬虫和禁用 JavaScript 的读者看到的都是完整内容。

这个主题没有用 `partialCached`：任何「缓存」的说法在这里都不成立。需要跨 partial 共享状态时就写进 `.Store`，需要缓存就自己用 `partialCached`，但先确认它真的省下了时间。
