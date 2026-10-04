+++
title = '一个站点的两种语言'
description = '双语站点是怎么接起来的：.en.md 配对、语言切换器的回退、hreflang 与 og:locale 的差异，以及按语言分的菜单和日期格式。'
date = 2026-03-24
authors = ['Hencter Lew']
tags = ['Hugo', '多语言']
categories = ['工程实践']
series = ['从零搭一个 Hugo 站点']
+++

## 配对规则：`.en.md` 而不是两棵目录树

这个站点只维护**一棵内容树**。中文是默认语言，直接占着文件名；
英文是同目录下同名加 `.en` 后缀的那份：

```text
content/
├── _index.md / _index.en.md                    # / 与 /en/
├── about/index.md / about/index.en.md          # /about/ 与 /en/about/
└── blog/hello/index.md / blog/hello/index.en.md # /blog/hello/ 与 /en/blog/hello/
```

Hugo 的配对依据是「同一目录、同一个 base name、语言后缀不同」，与目录叫什么无关。
配对成功的标志是 `.Translations` 与 `.AllTranslations` 里有东西，
`lang-switcher.html` 就是靠查后者生成语言入口的。叶子包的 `index.md` 与分支包的
`_index.md` 是两类不同的页面，**同一个目录里不能同时放**。

## 另一种做法，以及为什么不能混用

另一种做法是按语言分目录：`content/zh-cn/**` 和 `content/en/**` 各一棵树，
每个语言在自己的配置里设一个 `contentDir`。内容量大、两边由不同的人维护、
翻译进度长期不同步的项目适合这一种。

问题出在混用：把 `content/zh-cn/legal/privacy.md` 和 `content/legal/privacy.en.md`
放进同一个仓库，Hugo 一个都配不上——它在每种语言的内容树里都只看见一份文件，
`.en.md` 后缀只在**两种语言共用同一棵树**时才是配对信号。于是每页的
`.AllTranslations` 长度都是 1，整站悄悄变成单语的，而且没有任何警告。

## 语言切换器为什么回退到首页

语言入口是最容易出现 404 的地方，所以 `lang-switcher.html` 从不直接拼 URL：

```go-html-template
{{- range hugo.Sites -}}
  {{- $url := .Home.RelPermalink -}}
  {{- with $page.Translations -}}
    {{- range . -}}
      {{- if eq .Language.Lang $lang }}{{ $url = .RelPermalink }}{{ end -}}
    {{- end -}}
  {{- end -}}
{{- end -}}
```

逻辑是「优先当前页面的译文，没有就退到该语言的首页」：`$url` 一开始就是对方语言的首页，
只有找到同语言的译文才会被覆盖。代价是在一篇还没翻译的文章上点切换，
读者会落在首页而不是对应的文章，页面本身不会提示。

## `hreflang` 与 `og:locale`：连字符对下划线

同一个语言文字有两条标签，格式不同：`hreflang` 是 BCP-47，用**连字符**（`zh-CN`）；
`og:locale` 是 Open Graph 的方言，用**下划线**（`zh_CN`）。
两者都来自同一个 `site.Language.Locale`，所以其中一边必须改写——
`layouts/_partials/head/opengraph.html` 里就是一次 `replace`：

```go-html-template
<meta property="og:locale" content="{{ replace site.Language.Locale "-" "_" }}">
```

`layouts/_partials/head/alternates.html` 直接输出 `Locale`，另外补一条 `x-default`。
后者只是「没有更合适的语言时给谁」，它**不跟语言顺序走**：加一种语言、改一次 `weight`，
顺序就变了，所以它从 `params.seo.xDefaultLang` 读，当前配置的值是 `zh-cn`。

## 菜单与日期格式都按语言分开写

菜单不走词条表。中英两份分别在 `config/_default/menus.zh-cn.toml` 和 `menus.en.toml` 里，
各自声明 `[[main]]`，用 `pageRef = '/docs'` 关联页面：菜单标签是导航结构而不是模板里的字符串，
缺一个 i18n 键会让每个页面都多一条警告，而英文站通常要换掉或重排条目，不是逐条翻译。

日期格式同样按语言给，写在 `config/_default/languages.toml` 的 `[<lang>.params]` 下面：

```toml
[zh-cn.params]
  dateFormat = '2006 年 1 月 2 日'

[en.params]
  dateFormat = 'January 2, 2006'
```

这里不能用 `:date_long` 这个本地化令牌：它的本地化数据并没有覆盖所有语言，
zh-CN 会回退到英文，于是中文页面上出现英文日期。

## 界面文案怎么进到 JavaScript

模板里的文案走 Hugo 的 i18n：`i18n/zh-cn.toml` 与 `i18n/en.toml` 逐键对应，
模板调用 `{{ T "searchNoResults" }}`，两边缺一个键时 `--printI18nWarnings` 会说话。
但搜索框、复制按钮的文案是 JavaScript 在运行时写的，模板渲染不到它们，
它们走 `js.Build` 的 `params`：`head/js.html` 把 `T` 的结果拼成一个 `i18n` 字典传下去，
esbuild 把它暴露成虚拟模块 `@params`，所以每种语言有自己的 bundle 和哈希。
