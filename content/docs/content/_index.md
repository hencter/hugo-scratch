+++
title = '内容'
linkTitle = '内容'
description = '内容目录怎么映射成 URL，一页的头信息、正文与资源文件各自由谁负责。'
weight = 30
+++

这一章讲的是 `content/` 目录本身：一个 Markdown 文件怎么写、放在哪里，才既能变成正确的 URL，又能让模板、搜索索引和机器可读输出都读到该读的东西。

## 一章四页，各管一件事

内容这一章只有四页，但顺序不是随便排的：

- **front matter** 决定这一页的身份——标题、日期、权重，以及本主题真正读取的那些字段；
- **page bundles** 决定资源放在哪里，为什么 `content/blog/hugo-pipes/cover.png` 要跟 `index.md` 放在同一个目录；
- 短代码与渲染钩子决定了 Markdown 之外的能力，它们分别在 [本章的短代码部分](/docs/content/shortcodes/) 与 [渲染钩子](/docs/content/render-hooks/) 中说明。

四页里没有一页是纯概念：每一页都指向这个仓库里真实存在的文件。

## 这个站点的内容契约只有一份

`config/_default/hugo.toml` 里没有任何按目录划分的 front matter 规则，站点用的是同一套字段：`title`、`linkTitle`、`description`、`date`、`weight`，加上文档页额外的 `difficulty`、`estimatedTime`、`prerequisites`、`outcomes`。这四组额外字段不是装饰——`layouts/_partials/facts.html` 把前三个渲染成页面顶部的信息面板，`layouts/home.pages.json` 把同一批值写进机器可读索引。

{{< note >}}
字段写在 front matter 里，面板与 `pages.json` 就不会各说各话。一旦把「预计用时 15 分钟」写进正文，两个输出就只剩一个是对的。
{{< /note >}}

## content/ 到 URL 的映射

| 文件 | URL | 类型 |
| --- | --- | --- |
| `content/_index.md` | `/` | 首页 |
| `content/docs/_index.md` | `/docs/` | 章节首页（branch bundle） |
| `content/docs/content/front-matter.md` | `/docs/content/front-matter/` | 普通页 |
| `content/blog/hugo-pipes/index.md` | `/blog/hugo-pipes/` | 叶子包（leaf bundle） |
| `content/about/index.md` | `/about/` | 叶子包 |

两条规则覆盖了全部情况：`_index.md` 生成章节页，其余 `.md` 生成普通页；文件名就是 slug，`index.md` 例外——它让所在目录本身成为页面。

## 中英双语靠文件名配对

简体中文写在 `front-matter.md`，英文写在同一个目录的 `front-matter.en.md`。Hugo 按「同目录、同主文件名、语言后缀」配对，于是 `.Translations`、`.AllTranslations`、语言切换器与 `sitemap.xml` 里的 `xhtml:link` 全都自动成立，不需要把 `content/` 拆成两棵目录树。

配对的前提是两份文件**结构一致**：标题层级、短代码调用、代码块都要对应。它们不是两份文档，而是同一页的两种语言。

## 接下来读哪一页

如果你刚克隆仓库、只想改一句话，先看 [快速开始](/docs/start/quick-start/)。如果你要新建一页并让它出现在侧栏里，从 [front matter](/docs/content/front-matter/) 读起——`weight` 决定它在侧栏和上下页导航中的位置，写错了页面不会消失，只会换一个位置。
