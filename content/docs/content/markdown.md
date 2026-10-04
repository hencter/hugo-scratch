+++
title = 'Markdown 与扩展语法'
linkTitle = 'Markdown'
description = '这个站点打开了哪些 Goldmark 扩展，以及每一处配置在渲染结果里长什么样。'
date = 2026-02-16
weight = 30
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/content/front-matter/']
outcomes = ['知道哪些 Markdown 写法被扩展过', '能读懂 config/_default/hugo.toml 里的 markup 段', '知道渲染钩子的作用范围']
tags = ['Markdown', 'Goldmark']
+++

这一页的每一段都是**活的**：左边是写法，右边是它在这个站点上真正渲染出来的结果。所有开关都在 `config/_default/hugo.toml` 的 `[markup]` 段里。

## 标题、锚点与目录

`##` 到 `###` 的标题会自动获得 id，并带一个悬停才出现的锚点链接——这两件事都发生在渲染钩子 `render-heading.html` 里，而不是 Markdown 本身。这个页面的右侧目录来自 `.Fragments.Headings`，只收录 `##` 和 `###`，因为 `[markup.tableOfContents]` 里写的是 `startLevel = 2`、`endLevel = 3`。

正文里不要再写一级标题：模板已经渲染了 `<h1>`，多一个会让页面的标题结构出现两个并列的顶层。

## 表格：对齐、属性与横向滚动

表格经过 `render-table.html`，会被包进 `.table-wrap`——窄屏上横向滚动的是表格，不是整页。

| 输出格式 | 模板文件 | 媒体类型 |
| --- | --- | --- |
| `md` | `layouts/page.md`、`layouts/list.md` | `text/markdown` |
| `llms` | `layouts/home.llms.txt` | `text/plain` |
| `search` | `layouts/home.search.json` | `application/json` |

冒号控制对齐：

| 左对齐 | 居中 | 右对齐 |
| :--- | :---: | ---: |
| a | b | c |
| 长一点的内容 | 长一点的内容 | 1 |

{.no-wrap-first-col}

最后那行 `{.no-wrap-first-col}` 是**块级属性**，它属于上面那张表，被 `[markup.goldmark.parser.attribute] block = true` 解析。关掉这一项，它会变成表格后面的一行普通文字——所以打开它是有代价的：表格之后紧跟独立花括号时要注意。

## 引用与 GitHub 风格提示块

普通引用：

> 文档里没写、报错又指向别处的坑，才是真正花时间的地方。

`> [!NOTE]` 这种写法会被 `render-blockquote.html` 转成和 `{{</* note */>}}` 短代码**同一套标记**的提示块：

> [!NOTE]
> 两个入口共用一个 partial，所以它们的外观不可能各走各的。

> [!WARNING]
> 五个提示词（NOTE / TIP / IMPORTANT / WARNING / CAUTION）映射到四种样式，映射表写在渲染钩子里，因为 `T "important"` 会是一条缺翻译。

## 代码块、文件名与复制按钮

围栏代码块经过 `render-codeblock.html`：它调用 `transform.HighlightCodeBlock` 做高亮，再自己加上语言标签、文件名和复制按钮。文件名来自围栏信息串里的选项：

```go-html-template {filename="layouts/_partials/head/css.html"}
{{- with resources.Get "css/design-system.css" -}}
  {{- $design := . | css.Build (dict "minify" false) -}}
{{- end -}}
```

行内代码是 `` `css.TailwindCSS` ``，渲染成 `css.TailwindCSS`。语言未知时 Chroma 退回纯文本，不会报错。

## 列表：任务、定义与嵌套

任务列表渲染成带复选框的 `<ul>`，它仍然是列表——`list-style` 由主题的 `base.css` 控制，因为 Tailwind 的 preflight 被显式跳过：

- [x] 渲染钩子接住坏链接
- [x] 渲染钩子接住坏图片
- [ ] 给图片补一个自动 srcset

定义列表的**术语**会自动获得 id（`autoDefinitionTermID = true`）：

Hugo
: 用 Go 写的静态站点生成器，本项目用 0.146 以上版本。

Goldmark
: Hugo 默认的 Markdown 渲染器，`[markup.goldmark]` 全部是它的配置。

## 数学公式：透传而不是渲染

`passthrough` 扩展把定界符原样交给前端渲染器：

$$ \int_{0}^{1} x^2 \, dx = \frac{1}{3} $$

行内的 \(a^2 + b^2 = c^2\) 也一样。这个主题**故意不加载** KaTeX 或 MathJax：那是一份体积可观的三方脚本，而从不写公式的站点不该为它付费。没有渲染器时读者看到的是 LaTeX 源码，这比看着像坏掉的标记要诚实。

## 原始 HTML、属性与表情

把 `[markup.goldmark.renderer] unsafe` 打开后，正文里的 HTML 会原样保留：

<div class="callout callout--tip">
  <p class="callout__title">直接写 HTML</p>
  <div class="callout__body">这是正文里手写的 HTML，没有经过短代码，也没有被转义。</div>
</div>

也正因为打开了它，`{{%/* tabs */%}}` 这类**Markdown 记法**的短代码才可用——它们的输出会被重新当作 Markdown 解析，关掉 `unsafe` 时整块会被替换成 `<!-- raw HTML omitted -->`。

行内属性可以加在标题上：

### 带自定义 id 的小节 {#custom-anchor}

写法是 `### 标题 {#custom-anchor}`，由渲染钩子里的 `.Attributes` 决定最终 id。表情由 `enableEmoji = true` 打开 :smile:。

## 排版替换（typographer）

`[markup.goldmark.extensions.typographer]` 打开后，直引号会变成弯引号，`--` 变成 –，`---` 变成 —。写内容时如果非要字面的直引号或连续短横，记得它们是会被替换的——这就是为什么这个仓库的配置把替换目标逐项写清，而不是留空。
