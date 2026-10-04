+++
title = '更新日志'
linkTitle = '更新日志'
description = '由 content/changelog/_content.gotmpl 从 data/changelog.toml 生成，两种语言读同一份数据。'
weight = 30
+++

## 唯一的数据源

这一节里的页面**没有一页是手写的**。所有的发行说明只存在于一个文件里：`data/changelog.toml`，
里面的每一条 `[[releases]]` 带 `version`、`date`、`title` / `title_en`
和 `notes` / `notes_en` 两个数组。改一条记录，两个语言的列表和页面会一起变。

这也解释了为什么这个文件里的每个文本字段都成对出现：中英两种读者看的是同一份数据，
而不是两份需要手工对齐的文档。数据源只有一处，就不存在「列表更新了、页面没更新」这种状态。

## 内容适配器在构建时做什么

`content/changelog/_content.gotmpl` 是一个**内容适配器**（content adapter）：
一个在构建期创建页面的 Go 模板。它遍历 `hugo.Data.changelog.releases`，
把每个版本交出去：

```go-html-template
{{- $slug := replace .version "." "-" -}}
{{- $notes := slice -}}
{{- range .notes }}{{ $notes = $notes | append (printf "- %s" .) }}{{ end -}}
{{- $body := printf "## %s\n\n%s\n" .title (delimit $notes "\n") -}}

{{- $.AddPage (dict
      "path"        $slug
      "kind"        "page"
      "title"       (printf "%s — %s" .version .title)
      "description" .title
      "date"        .date
      "params"      (dict "version" .version)
      "content"     (dict "mediaType" "text/markdown" "value" $body)) -}}
```

`path` 是相对这个内容目录的路径，不带扩展名；`.version` 里的点换成连字符，
所以 `v1.11.0` 落到 `/changelog/v1-11-0/`。`content` 给的是 `mediaType` + `value`，
正文就是普通 Markdown，渲染路径和别的页面完全一样——它照样会过渲染钩子，
也照样会输出自己的 `index.md` 孪生页。

这样做的收益很具体：**列表和页面不可能对不上**，因为两边是同一次渲染的两个投影；
新增一个版本是往 TOML 里追加一条，而不是新建一个目录再记得去改列表。

## 它做不到的事

内容适配器只在**默认语言**里创建页面。给 `path` 加语言后缀不会产生一份译文，
只会产生一个 URL 里带着那个后缀的页面——`/changelog/v1-0-0.en/` 是一个独立页面，
不是 `/changelog/v1-0-0/` 的英文版。所以这一节里 `.Translations` 是空的，
语言切换器在这类页面上会退回首页。

英文那一侧因此走另一条路：同一份数据由 `{{< changelog >}}` 短代码在
`_index.en.md` 里内联渲染，读者看到的是同样的发行历史，只是它的载体是
英文版章节页，而不是一组「英语」的生成页面。

> [!WARNING]
> 数据文件里的 `notes` 是按行使用 Markdown 的：每一条会变成列表中的一个 `- ` 项。
> 在一条 note 里写多行文本不会变成多段，需要换行就拆成两条。

## 分页与版本号

`config/_default/hugo.toml` 里 `pagination.pagerSize` 是 **2**——这个值对真实站点来说太小，
放在这里是为了让分页控件在演示站点里真的出现。所以这一节会分成多页，
`layouts/_partials/pagination.html` 的窗口分支也因此有机会跑到。

版本号的写法沿用 `v<major>.<minor>.<patch>`，和主题仓库的标签一致：
每个版本对应主题与站点各自的一次提交，`data/changelog.toml` 里的 `date`
就是那次提交的日期。
