+++
title = '结构化数据'
linkTitle = '结构化数据'
description = '一个 JSON-LD @graph 里的四个节点各写什么、字符串字面量陷阱为什么会让所有消费者拒绝它、面包屑如何与可见导航保持一致，以及验证方式。'
date = 2026-02-25
weight = 20
difficulty = 'advanced'
estimatedTime = 18
prerequisites = ['/docs/', '/docs/seo/', '/docs/seo/semantic-html/']
outcomes = ['逐个节点说出 schema.html 生成了什么', '解释 jsonify 的字符串陷阱和 safeJS 修法', '用两个官方工具验证一页的结构化数据']
tags = ['SEO']
+++

结构化数据是这一章里唯一"看不见也猜不出来"的部分：页面上没有任何变化，输出里却多了一段 JSON。它由 `layouts/_partials/head/schema.html` 一个文件生成，长度过百行，但真正需要理解的决定只有四个。

## 一个 @graph，而不是十个 script

模板只输出一个 `<script type="application/ld+json">`，里面是一个 `@graph` 数组。用图而不是用多个脚本，原因在节点之间的引用：Organization 有自己的 `@id`（首页地址加 `#organization`），WebSite 节点通过 `publisher` 指向那个 `@id`，页面节点又通过 `isPartOf` 指向 WebSite 的 `@id`。消费者拿到一段 JSON 就能把发布者、站点和当前页面全部还原出来，不需要再发第二次请求。

`@id` 用绝对地址构造，是因为它必须跨页面稳定：同一份 Organization 节点出现在全站每一页上，搜索引擎才知道它们是同一个实体，而不是几千个同名的组织。站点标题和 `params.seo.organization` 决定 `name`，`params.seo.logo` 经与 Open Graph 共用的图片解析后写成 `ImageObject`。

## 四个节点各是什么

**Organization** 描述发布者：`@id`、`name`、`url`，以及可选的形象图片。站点没有填 `organization` 时回落到 `site.Title`，所以这一段永远不会缺字段。

**WebSite** 描述站点本身：`@id`、`url`、`name`、`inLanguage`（取 `site.Language.Locale`）和 `publisher`。首页存在 `search` 输出格式时，它会额外得到一个 `potentialAction`，即 `SearchAction`：`target.urlTemplate` 指向 `search?q={search_term_string}`，`query-input` 声明参数名。这段是条件生成的——把 `search` 从 `[outputs]` 里删掉，搜索动作也随之消失，不会留下一个指向空页面的声明。

**页面节点**是本页的主角，类型由页面种类决定：

- 首页 → `WebSite`；
- 其他分支页（章节、分类、标签）→ `CollectionPage`；
- 普通页面 → `WebPage`；
- `blog` 章节下且带日期的页面 → `BlogPosting`。

其余字段从页面本身取：`url` 与 `name`、`isPartOf`、`inLanguage`、`description`（会先 `plainify` 去掉标记）、`datePublished` 与 `dateModified`（RFC 3339 形式）、`wordCount`、`timeRequired`、作者与配图。两处细节值得一提：`timeRequired` 写成 `PT<n>M` 而不是一个数字，且最少算一分钟，因为 `ReadingTime` 是整数，直接参与 `math.Max` 会变成浮点数，打印出来就是一段乱码；`keywords` 读的是 front matter 的 `keywords`，不是 `tags`——标签有自己的分类页面，关键词只是给这一页用的。

**BreadcrumbList** 是第四个节点，放在最后生成，因为它要等前三者。它的每一项都是 `ListItem`：位置从 1 开始，名称和链接来自 `.Ancestors.Reverse`，最后补上当前页面自己。

## 字符串字面量陷阱与它的修法

这段模板最值得单独记住的一行是最后那行输出。正确的写法是把一个**对象**交给 `jsonify`，再把结果按 JS 插入：

```text
{{ $json := dict "@context" "https://schema.org" "@graph" $graph | jsonify }}
<script type="application/ld+json">{{ $json | safeJS }}</script>
```

如果反过来，把已经渲染好的字符串再 `jsonify` 一次——例如把某个 partial 的返回值直接管道给 `jsonify`——得到的是一个 JSON **字符串字面量**：整段 `@graph` 会被引号包住、内部引号被转义，所有消费者都只看到一个字符串，于是"结构化数据无效"。这不是 Hugo 的怪癖，`jsonify` 的工作就是把值编码成 JSON，字符串的 JSON 编码本来就是一个带引号的字符串。

第二个陷阱是转义。Go 的 JSON 编码器会把 `<`、`>`、`&` 写成 `\u003c`、`\u003e`、`\u0026`，所以负载里出现 `</script>` 也无法提前闭合脚本元素；模板末尾还额外把 `</` 替换成 `<\/`，理由是即便将来 Hugo 不再做这层转义，行为也不会退化。两重保险的结果是：这段 JSON 可以安全地放进 HTML，也可以被任何标准解析器读出。

还有一个前置条件让头部的模板能安全读取正文信息：`layouts/_partials/layout/flags.html` 会访问 `$page.Fragments`，而它由 `baseof.html` 在渲染 `head.html` **之前**调用。因此 `.WordCount`、`.ReadingTime` 和标题结构在写 `<head>` 时已经就绪，`wordCount` 和 `timeRequired` 才有了数据来源。

## BreadcrumbList 必须和可见面包屑一致

`layouts/_partials/breadcrumbs.html` 在页面上渲染一条 `<ol>`，`schema.html` 生成对应的 `BreadcrumbList`，两者读的是同一份 `.Ancestors.Reverse`。这不是巧合，而是要求：可见面包屑和结构化面包屑不一致属于结构化数据错误，搜索引擎会按"页面上的导航和声明不符"来处理。

因此改动这一层时不要只改一边。往可见路径里加一级（比如在页面标题前插入一个分类链接），就要在 JSON-LD 里同时体现；反过来，如果某个页面结构上不该出现在面包屑里，正确的做法是让它别出现在 `.Ancestors` 中，而不是在 JSON-LD 里手工跳过。

## 怎么验证

先看原始输出，再交给官方工具——顺序不要反，因为工具报的错往往指向工具没读到数据，而不是数据写错了：

```bash
hugo --ignoreCache
grep -o '"@type":"[^"]*"' public/docs/seo/structured-data/index.html
```

你应该看到 `Organization`、`WebSite`、`WebPage`（或本节对应的 `CollectionPage`）和 `BreadcrumbList` 各一次。若一个都没有，说明 `schema.html` 没被包含——检查 `layouts/_partials/head.html` 里的调用顺序，它是最后一个头部 partial。

{{% steps %}}
{{% step "看本地输出" %}}
构建之后打开 `public/` 下任一页面的 HTML，搜索 `application/ld+json`，把那段 JSON 复制到一个能报错的编辑器里。语法错误在这一步就能发现，不必等工具判"无效"。
{{% /step %}}
{{% step "schema.org 验证器" %}}
把页面地址或那段 JSON 贴进 [validator.schema.org](https://validator.schema.org/)，它检查的是词汇表和字段类型，会指出"这个属性不属于这个类型"这一类问题。
{{% /step %}}
{{% step "Google 富媒体测试" %}}
再用 [Rich Results Test](https://search.google.com/test/rich-results) 看 Google 实际会从中抽出什么。它只关心它支持的富媒体类型，所以"没有富媒体结果"不等于数据有错，两者要分开看。
{{% /step %}}
{{% /steps %}}

{{< warning >}}
验证工具需要能访问到页面。本地 `hugo server` 的地址通常无法被抓取，而预览环境又带着 `noindex, nofollow`——这不会阻止 JSON-LD 被解析，但会让工具看不到内容。最省事的做法是在一个临时公开的部署上验证，或者直接把 JSON 贴进 schema.org 的验证器。
{{< /warning >}}
