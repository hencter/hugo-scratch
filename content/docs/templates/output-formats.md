+++
title = '输出格式'
linkTitle = '输出格式'
description = '同一份内容为什么同时产出 index.html、index.md、llms.txt 与 pages.json，以及这些文件名由谁决定。'
date = 2026-02-20
weight = 20
difficulty = 'advanced'
estimatedTime = 25
prerequisites = ['/docs/templates/', '/docs/templates/templates/']
outcomes = ['说清主题声明格式、站点决定输出的分工', '为一种新格式写出正确的模板文件名', '读懂 md 输出里的 transform.Remarshal']
tags = ['Hugo']
+++

一个页面不必只产出 HTML。这个站点给首页额外产出 Markdown、纯文本与两份 JSON，给每个页面产出一份 Markdown 副本。这一页讲清是谁在做这件事，以及为什么文件名叫 `home.llms.txt`。

## 主题声明格式，站点决定谁产出

`themes/hugo-scratch-theme/hugo.toml` 里有媒体类型与四份格式声明：

```toml
[mediatypes.'text/markdown']
  suffixes = ['md']

[outputformats.md]
  mediaType      = 'text/markdown'
  baseName       = 'index'
  isPlainText    = true
  isHTML         = false
```

主题的配置文件只能设置 `params`、`menu`、`outputformats` 与 `mediatypes`，`[outputs]` 不在其中。于是分工很干净：**格式长什么样由主题决定，哪些页面产出它由站点决定**。`config/_default/hugo.toml` 里是这样写的：

```toml
[outputs]
  home = ['html', 'rss', 'md', 'llms', 'search', 'pages']
  section = ['html', 'rss', 'md']
  taxonomy = ['html', 'rss', 'md']
  term = ['html', 'rss', 'md']
  page = ['html', 'md']
```

从列表里删掉一个名字，对应的文件就不再生成——`layouts/_partials/footer.html` 里的链接来自 `.OutputFormats`，所以链接会跟着一起消失，不会留下死链。

## 文件名是怎么拼出来的

非 HTML 格式的模板名是 `{页面类型模板名}.{格式名}.{后缀}`。查找 `home.llms.txt` 时，Hugo 先找 `layouts/home.llms.txt`，再退回 `layouts/_default/home.llms.txt` 之类的位置。这个主题的文件名与产出的地址一一对应：

| 模板 | 产出 | 说明 |
| --- | --- | --- |
| `layouts/home.llms.txt` | `/llms.txt` | `baseName = 'llms'` |
| `layouts/home.search.json` | `/search.json` | 站内搜索的索引 |
| `layouts/home.pages.json` | `/pages.json` | 机器可读的页面清单 |
| `layouts/list.md` | 列表页的 `index.md` | 章节、分类法、词条 |
| `layouts/page.md` | 普通页的 `index.md` | 每个页面一份 |
| `layouts/rss.xml` | `index.xml` | `rss` 格式 |
| `layouts/sitemap.xml` | `sitemap.xml` | 覆盖 Hugo 内置模板 |
| `layouts/robots.txt` | `/robots.txt` | 由 `enableRobotsTXT` 打开 |

`md` 格式的 `baseName` 是 `index`，所以 Markdown 副本不是 `page.md` 而是 `index.md`，跟 `index.html` 并排放在同一个目录里——在任意 URL 后面加 `index.md` 就能读到它的 Markdown 版本。

{{< note >}}
本主题的模板根目录用的是当前模板系统：`baseof.html`、`home.html`、`page.html`，没有 `_default/`。列表类的 HTML 由 `layouts/section.html` 渲染，而它的 Markdown 孪生页由 `layouts/list.md` 渲染——**同一个页面的两种输出，用的是两个不同名字的模板**，这是初学时最容易找错的地方。
{{< /note >}}

## `isPlainText` 与 `notAlternative`

`isPlainText = true` 让 Hugo 用 `text/template` 而不是 `html/template` 渲染这个格式，输出不做 HTML 转义。Markdown 与 JSON 都必须这样：不设它，`#` 与 `<` 会被转义成实体，产出的 Markdown 就是坏的。

`notAlternative = true` 则把它排除在 `.AlternativeOutputFormats` 之外。`head/alternates.html` 用这个集合生成 `<link rel="alternate">`：

- `rss` 与 `md` 没有设它，所以每页都会公告 `/index.md` 与 `/index.xml` 这两个替代版本；
- `llms`、`search`、`pages` 都设了，它们只挂在首页，硬塞进每一页的 `<head>` 只会是噪音。

## 首页多语言站点地图是个索引

这个站点有两种 sitemap 产物，容易只看到其中一种：

- `layouts/sitemap.xml` 覆盖内置模板，为每个语言站点各生成一份 `/zh-cn/sitemap.xml` 与 `/en/sitemap.xml`，逐页读取 `.Sitemap.ChangeFreq` 与 `.Sitemap.Priority`，并用 `xhtml:link` 把互为译文的页面串起来；
- 多语言站点的根目录还会多出一份 `/sitemap.xml`，它是 **sitemapindex**，只列出上面那两份的地址。

实测的根站点地图就是这样：

```xml
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap><loc>https://example.com/zh-cn/sitemap.xml</loc></sitemap>
  <sitemap><loc>https://example.com/en/sitemap.xml</loc></sitemap>
</sitemapindex>
```

`robots.txt` 里的 `Sitemap:` 必须指向绝对地址，而爬虫顺着索引就能找到每个语言的完整清单。

## Markdown 孪生页的 front matter

`layouts/page.md` 与 `layouts/list.md` 用 `transform.Remarshal "yaml"` 输出 YAML front matter。手写 `key: "{{ .Title }}"` 在标题里出现引号的那一天就会坏，而先把数据组装成 `dict`、再交给 `Remarshal` 序列化，输出永远是合法 YAML：

```go-html-template
{{- $meta := dict "title" .Title "url" .Permalink "tags" $tags -}}
---
{{ $meta | transform.Remarshal "yaml" }}---
```

`page.md` 还会把 `difficulty`、`estimatedTime`、`prerequisites`、`outcomes` 一并写进这份头信息，与页面上的信息面板同源；`translations` 字段列出这一页每个语言版本的绝对地址。

## 改完怎么验证

`hugo --ignoreCache` 之后直接看产物，比读模板快：

- `public/index.md`、`public/llms.txt`、`public/search.json`、`public/pages.json`、`public/index.xml`、`public/sitemap.xml` 六个文件应当都在；
- 任意页面的 `public/docs/.../index.md` 开头必须是合法的 YAML，而不是带转义的字符串；
- `public/sitemap.xml` 是索引，`public/zh-cn/sitemap.xml` 才是逐页清单。

想再加一种格式，顺序是：在主题里声明 `outputformats`，在站点 `[outputs]` 里挂到某类页面上，然后写一个 `{kind}.{name}.{suffix}` 模板。三处缺一，产物都不会出现。
