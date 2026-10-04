+++
title = '页面头信息（front matter）'
linkTitle = '页面头信息'
description = '这个主题真正读取的字段、TOML 的表格陷阱，以及日期为什么来自 Git 提交。'
date = 2026-02-20
weight = 20
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/content/']
outcomes = ['写出这个主题能读懂的 front matter', '让 lastmod 由 Git 提交历史决定', '用 build 选项控制一页是否列出、是否渲染']
tags = ['Hugo']
+++

每页开头两个 `+++` 之间的部分是 front matter。它决定了这一页的身份（URL 之外的标题、日期、排序），也决定了模板能读到什么。这一页只讲这个站点实际使用的字段，以及几个会静默出错的写法。

## TOML、YAML 还是 JSON

Hugo 认得三种格式，靠开头那一行区分：`+++` 是 TOML，`---` 是 YAML，`{` 开头是 JSON：

```toml
+++
title = '快速开始'
weight = 10
+++
```

```yaml
---
title: 快速开始
weight: 10
---
```

```json
{ "title": "快速开始", "weight": 10 }
```

这个仓库全线使用 TOML：`content/` 下每一页、`config/_default/*.toml`、`data/changelog.toml` 都是 TOML，所以读到 `+++` 就知道是同一套语法。三种格式里只有 TOML 有一个必须记住的陷阱——JSON 与 YAML 都没有表格，也就没有这个问题。

{{< warning "TOML 的表格陷阱" >}}
一个裸键写在表格头下面，就属于那张表。`config/_default/hugo.toml` 顶部的注释正是为这件事写的：

```toml
[frontmatter]
  lastmod = [':git', 'lastmod', 'date']
theme = ['hugo-scratch-theme']   # 错：这一行变成了 frontmatter.theme
```

后果不是语法错误，而是 `theme` 悄悄失效——站点照样构建，只是找不到任何布局，报错只说 "found no layout file for kind"。所有标量键必须写在第一个 `[table]` 之前。同样的坑在页面里也能踩到：

```toml
+++
title = '仅索引条目'
+++

[build]
  render = 'never'
weight = 20   # 错：变成了 build.weight，页面排序还是默认值
```
{{< /warning >}}

## 这个主题读取的字段

| 字段 | 类型 | 谁在读它 |
| --- | --- | --- |
| `title` | string | `<h1>`、`<title>`、Open Graph、`pages.json` |
| `linkTitle` | string | 侧栏、面包屑、卡片、上一页/下一页 |
| `description` | string | `<meta name="description">`、列表摘要 |
| `date` | date | `<time datetime>`、排序、`sitemap.xml` 的 `lastmod` 起点 |
| `lastmod` | date | 「更新于」一行、结构化数据的 `dateModified` |
| `weight` | int | 侧栏与同章节内的排序 |
| `difficulty` | string | `layouts/_partials/facts.html` 的信息面板 |
| `estimatedTime` | int | 同一面板，显示为分钟 |
| `prerequisites` | []string | 面板中的链接；值是站内路径 |
| `outcomes` | []string | 面板中的「你将学会」列表 |
| `tags` / `categories` / `series` | []string | 三个分类法页面 |
| `images` | []string | `og:image` 与 JSON-LD 的 `image` |
| `notice` | string | `layouts/_partials/banner.html` 顶部的提示条 |
| `toc` | bool | 设为 `false` 时本页不出目录 |
| `sitemap` | table | `changefreq` / `priority`，被 `layouts/sitemap.xml` 读取 |

`linkTitle` 是唯一一个「不写也能跑、写了才顺手」的字段：侧栏列的是它，回退到 `title`；中文标题很长时，它决定栏里那一行折几行。`weight` 只需要在**自己的章节里唯一**：这一章各页是 10、20、30，间隔 10，中间插一页就填 15，不必重排后面的。

## 日期：字段、Git 与 `[frontmatter]` 映射

`config/_default/hugo.toml` 里两件事一起生效：

```toml
enableGitInfo = true

[frontmatter]
  lastmod = [':git', 'lastmod', 'date']
  date = ['date', ':git']
```

`enableGitInfo = true` 让 Hugo 从仓库里读出每一页的 `.GitInfo`（提交哈希、作者、提交主题）。`[frontmatter]` 里的列表是**回退顺序**：取 `lastmod` 时先问 Git，再问页面的 `lastmod`，最后退回 `date`。

观察到的结果最能说明问题：`content/legal/privacy.md` 的 `date` 是 2026-01-05，而构建产物里这一页的 `dateModified` 是最后一次提交的时间，页面上因此多出「更新于」一行——`layouts/_partials/page-meta.html` 只在 `.Lastmod` 与 `.Date` 不在同一天时才渲染这一行。也就是说，`lastmod` 不必手写，改一次文件、提交一次就更新一次。

{{< tip >}}
模板里判断日期必须用 `.IsZero`，不能用 `with` 去判断 `.Date` 是否存在：`.Date` 是 `time.Time` 结构体，永远为真，没有日期的页面会打印 `0001-01-01`。
{{< /tip >}}

## `build`：让一页只存在于索引里

需要「列出来但不渲染成页面」或「渲染但不出现在任何列表里」时，用 `build` 表：

```toml
[build]
  list = 'always'
  render = 'never'
  publishResources = false
```

`list` 取 `always` / `local` / `never`，`render` 取 `always` / `link` / `never`，只有 `publishResources` 是布尔值。键名是 `build`——旧写法 `_build` 已经废弃，用它会得到一条弃用告警，在 `--panicOnWarning` 下直接变成构建失败。

## 别名与级联

`aliases = ['/docs/front-matter/']` 会给旧地址生成一个跳转页，页面改名时用它保住入链。要批量给一组页面设置值，不要复制粘贴，用站点级的 `cascade`。`config/_default/hugo.toml` 里对博客和法律章节各有一条：

```toml
[[cascade]]
  [cascade.sitemap]
    changefreq = 'yearly'
    priority = 0.2
  [cascade.target]
    path = '/legal/**'
```

`cascade.target` 负责选页面（`path`、`kind`、`lang` 等），其余键合并进被选中页面的 front matter。构建产物里 `/legal/privacy/` 的 `changefreq` 是 `yearly`、`priority` 是 `0.2`，其余页面保持 `[sitemap]` 的 `weekly` / `0.5`——这句话是可验证的，不是推测。旧写法 `cascade._target` 自 0.156 起废弃。

改完一页，跑一次 `hugo --ignoreCache`，再去 `public/` 里读那一页——front matter 的问题几乎都能在渲染结果里看出来，而不是在构建日志里。[^1]

[^1]: 上游文档：[Front matter](https://hugozh.cn/content-management/front-matter/)
