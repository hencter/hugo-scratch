+++
title = '发现入口'
linkTitle = '发现入口'
description = 'robots.txt、多语言 sitemap、canonical 与 hreflang、每页的 Markdown 孪生、llms.txt、pages.json、search.json，以及站长验证。'
date = 2026-02-25
weight = 30
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/', '/docs/seo/']
outcomes = ['说清哪些文件是构建生成的、哪些是静态的', '知道多语言站点地图和 hreflang 的关系', '找到并验证三个机器可读入口']
tags = ['SEO']
+++

前面两页讲的是"一页写得好不好"，这一页讲"别人怎么找到这一页"。所有入口都是构建的产物，没有一个是手工维护的清单——正因为如此，删掉一个输出格式就会同时删掉对应的文件、页脚链接和站点地图条目，不会留下指向空地址的说明。

## robots.txt 是生成的，不是静态文件

静态站点通常把 `robots.txt` 放进 `static/`，这里没有：它由 `layouts/robots.txt` 在构建时生成，`config/_default/hugo.toml` 里的 `enableRobotsTXT = true` 打开这个行为。两个分支差别很大：

- 生产构建输出 `User-agent: *` 加 `Allow: /`，以及一行 `Sitemap:`。这一行必须是**绝对地址**，由 `"sitemap.xml" | absURL` 生成；robots.txt 的规范要求如此，相对路径会被爬虫安静地忽略。
- 非生产构建输出 `Disallow: /`，并在注释里写上是哪个环境。预览部署因此没有任何机会被收录，也就不会和正式站点争抢同一个索引。

判断依据是 `hugo.IsProduction`，而普通的 `hugo` 命令就是生产环境——这一点在[本章概览](/docs/seo/)里已经提过，但它在这里的后果更直接：在本地用 `hugo server` 看 `robots.txt`，你看到的永远是那一份禁止抓取的版本。

## sitemap.xml：多语言与逐页权重

`layouts/sitemap.xml` 覆盖了 Hugo 内置的实现，有三处不同。第一，每个 `<url>` 如果存在翻译，就输出一组 `<xhtml:link rel="alternate" hreflang="…">`，这是在告诉搜索引擎"这些是同一页的不同语言版本"，而不是彼此重复的内容。第二，`<changefreq>` 和 `<priority>` 来自**每一页自己的** `.Sitemap.ChangeFreq` / `.Sitemap.Priority`，所以 `config/_default/hugo.toml` 里那些 `[[cascade]]` 块才能真正落到文件里：`/blog/**` 是 `daily` 加 `0.9`，`/legal/**` 是 `yearly` 加 `0.2`，其余页面用 `[sitemap]` 的默认值 `weekly` 和 `0.5`。第三，页面集合先被防御式地取一次（`.Pages`，取不到再试 `.Data.Pages`），然后**断言非空**——空站点地图比构建失败糟糕得多，所以这里选择直接报错。

多语言站点在根目录还会多出一份索引：`/sitemap.xml` 是一个 `<sitemapindex>`，里面列出每个语言自己的站点地图，例如 `/en/sitemap.xml`。搜索引擎先读索引再读各自的列表，所以新增一种语言不需要改动任何模板。

## canonical、hreflang 与 x-default

`layouts/_partials/head/meta.html` 负责 canonical，它直接使用 `.Permalink`，因此永远是绝对地址。分页归档是一个容易被忽略的例外：第 2 页起的 canonical 指向分页自己的地址，而不是第 1 页——否则第 2 到第 n 页都会声明自己是第 1 页的副本，等于主动放弃收录。单页需要指向别处时，front matter 的 `canonical` 可以覆盖。

翻译互链写在 `layouts/_partials/head/alternates.html` 里，同一个 partial 还输出另一种 `rel="alternate"`：站点地图、RSS 和每页的 Markdown 孪生，都作为"本页的其他表示形式"声明。hreflang 循环的是 `.AllTranslations`（它总是包含当前语言），取值用 `.Language.Locale`，也就是 `zh-CN` 和 `en-US` 这样的 BCP-47 标签。

`x-default` 单独挑一次，指向 `params.seo.xDefaultLang` 指定的那个语言——这里是 `zh-cn`。它不能靠语言顺序猜：加一门语言、或者调整 `languages.toml` 里的 `weight`，猜出来的结果就会变，而 `x-default` 是给"没有匹配到任何语言"的访问者用的落地页，应该是一个明确的决定。

## 每页一个 Markdown 孪生，外加三个机器可读入口

`themes/hugo-scratch-theme/hugo.toml` 里声明了一个 `md` 输出格式：媒体类型 `text/markdown`、`baseName = 'index'`、`isPlainText = true`、`isHTML = false`；站点的 `[outputs]`（见[站点配置](/docs/configuration/site-config/)）再决定哪些页面产出它（页面是 `['html', 'md']`，章节、分类和标签同样有）。于是任一页面的地址后面接上 `index.md` 就是它的 Markdown 版本，例如 `/docs/assets/css/index.md`。

页面底部那个"Markdown 源文件"链接由 `layouts/page.html` 用 `.OutputFormats.Get "md"` 得到，同样不是手写的地址。它和 `rel="alternate"` 是同一份数据的两种呈现。

除了 HTML 和它的孪生，站点还发布三份给机器看的文件，全部来自首页的输出格式，声明在 `themes/hugo-scratch-theme/hugo.toml`：

- **`/llms.txt`**（`llms` 格式，`notAlternative = true`）：纯文本，先说明站点是什么，再按章节列出文档、最近的文章、机器可读文件的清单和全部页面。它存在的意义是"一次请求交代清楚"，适合语言模型或不带 HTML 解析器的客户端。
- **`/pages.json`**（`pages` 格式）：每页一条记录，带着 `role`、`title`、`url`、`markdown`、`tags`、`wordCount`、`readingTime`、翻译链接，以及 front matter 里的 `difficulty`、`estimatedTime`、`prerequisites`、`outcomes`。它回答的是"这一页该怎么用"，所以和 HTML 页面里的信息面板来自同一批字段，不会各说一套。
- **`/search.json`**（`search` 格式）：客户端搜索的索引，由 `assets/js/modules/search.js` 在读者第一次打开搜索框时取用；地址通过 `@params` 注入脚本，理由见 [JavaScript 管线](/docs/assets/js/)。每条记录的 `body` 取自 `.RawContent` 而不是渲染后的正文，因为渲染结果里混着标题锚点、复制按钮和代码语言标签，那些都不是作者写的字。

`layouts/_partials/footer.html` 把这几份文件和站点地图、RSS 一起列在页脚，并且是从 `.OutputFormats` 里取地址的——输出格式没了，链接也跟着消失。

## 站长验证

两件事都是可选的，都不影响上面任何入口：把站点登记到 Google Search Console 或 Bing Webmaster Tools，然后证明你是它的主人。模板 `layouts/_partials/head/verification.html` 读 `config/_default/params.toml` 里的 `[params.verification]`，填了 `google` 就输出 `google-site-verification` 元标签，填了 `bing` 就输出 `msvalidate.01`；两个都空着时**一个标签都不输出**，所以默认页面里没有死元数据。仓库里这两个键是注释掉的，要用就取消注释并贴上自己的值。

不想改配置也行：验证服务通常还提供"HTML 文件"方式，把下载到的文件原样放进 `static/`，Hugo 会把它拷到站点根目录。

## 一次自查清单

```bash
hugo --ignoreCache
head -5 public/robots.txt
head -20 public/sitemap.xml
head -30 public/pages.json
```

按顺序看四件事：`robots.txt` 里的 `Sitemap:` 是绝对地址；根 `sitemap.xml` 是索引且每个语言都列到了；`pages.json` 里每页都有 `url` 和 `markdown`；`search.json` 里能搜到你刚写的这一页。最后打开任一页面源码，确认 canonical、成对的 hreflang 和一段 JSON-LD 都在——四件事全过，站点对机器就是"可发现、可解析、可引用"的。

## 参考

{{< docref "templates/sitemap/"  >}}
