+++
title = '站点配置'
linkTitle = '站点配置'
description = 'config/_default 下的四个文件各自管什么，三层配置怎么合并，以及那个让站点静默失去主题的 TOML 陷阱。'
date = 2026-01-14
weight = 10
difficulty = 'intermediate'
estimatedTime = 20
prerequisites = ['/docs/configuration/']
outcomes = ['说出 config/_default 下每个文件的职责', '在正确的层里修改一项设置', '避开裸键写在 [table] 下面这个坑']
tags = ['Hugo', '配置']
+++

配置是在本地改得最多、出问题也最难定位的一层。原因是它没有页面级的报错：写错一个键的位置，Hugo 不会说"你写错了"，它只是按自己理解的那份配置去构建，然后让别的部件以奇怪的方式失败。这一页把三层合并关系、四个配置文件的边界，以及一个真实踩过的 TOML 陷阱讲清楚。

## 为什么不放在根目录

站点根目录没有 `hugo.toml`。配置全部在 `config/` 下，`_default` 放通用部分，环境名目录放环境特化部分。这么做换来三件事：主配置、语言、参数、菜单各自独立成文件，改一项不用通读全文；生产环境的差异有地方可放，不必在命令行上追加参数；主题与站点的覆盖关系变成文件级的事实，而不是"我记得主题里也有一份"。

根目录单文件与 `config/` 目录同时存在时，Hugo 两个都会读。既然能只留一种布局，就没有理由留两种。

## 合并顺序：主题 → _default → 环境

构建时配置按固定顺序叠加，后面的覆盖前面的：

{{% steps %}}
{{% step "主题配置" %}}
`themes/hugo-scratch-theme/hugo.toml`。主题配置只能提供 `params`、`menu`、`outputformats`、`mediatypes` 四类内容，文件里其它顶层键会被 Hugo 忽略。它提供的是"站点什么都不配也能构建"的地板值：`description`、`dateFormat = ':date_long'`、`[params.ui]` 下的一批显示开关、`[params.nav] sidebarSections = ['docs']` 等等。
{{% /step %}}
{{% step "站点默认配置" %}}
`config/_default/` 下的所有 `.toml` 文件。它们压过主题的同名键，`_default` 内部的文件之间则互不覆盖——`hugo.toml`、`languages.toml`、`params.toml` 各自负责不同的树。
{{% /step %}}
{{% step "环境配置" %}}
`config/production/hugo.toml`。一次普通的 `hugo` 构建运行在生产环境，因此这一层会被读入，优先级最高。`hugo server` 默认运行在开发环境，所以本地预览看到的是没有这一层的配置——这正是它存在的意义。
{{% /step %}}
{{% /steps %}}

判断一个键该写在哪一层，用一条经验规则：与具体部署环境无关的写 `_default`，只在正式发布时成立的写环境目录。比如站点描述、导航参数属于前者；只有上线才需要的统计脚本或更严格的输出设置属于后者。

## 四个文件的分工

**`config/_default/hugo.toml`** 管站点身份和构建行为。它声明了 `baseURL = 'https://hencter.github.io/hugo-scratch/'`、`title`、`locale = 'zh-CN'`、`defaultContentLanguage = 'zh-cn'`、`defaultContentLanguageInSubdir = false`（所以中文在根路径，英文在 `/en/` 下）、`enableGitInfo`、`hasCJKLanguage`、`timeZone = 'Asia/Shanghai'`，以及 `theme = ['hugo-scratch-theme']`。再往下是 `[module]` 挂载、`[frontmatter]` 日期来源、`[markup]`、`[taxonomies]`、`[pagination]`、`[outputs]`、`[cascade]` 等表。

**`config/_default/languages.toml`** 管语言。两个语言条目 `zh-cn` 与 `en` 各自声明 `label`、`locale`、`weight`、`title`，并在 `[<lang>.params]` 里给出显式的 `dateFormat`——不依赖 `:date_long` 这样的本地化记号，因为本地化数据并不覆盖所有语言。

**`config/_default/params.toml`** 管模板读取的参数：`description`、`tagline`、`author`、`repoURL`、`images`，以及 `[ui]`（`showBreadcrumbs`、`showTableOfContents`、`tocMinHeadings`、`showPrevNext` …）、`[nav] sidebarSections`、`[home] latestCount`、`[seo]`、`[analytics]` 等表。主题为每个键准备了默认值，所以这里的条目是覆盖，不是必需。

**`config/_default/menus.zh-cn.toml` 与 `menus.en.toml`** 管导航。菜单不放进 `params.toml` 也不是通过 i18n 目录翻译，原因见[导航](/docs/configuration/navigation/)。

## 经典错误：裸键落在表头下面

TOML 规定，裸键属于上方最近的那个表头。下面这段配置看起来完全正常：

```toml
[frontmatter]
  lastmod = [':git', 'lastmod', 'date']

theme = ['hugo-scratch-theme']
```

实际上 `theme` 的全名是 `frontmatter.theme`。Hugo 不会报配置错误：它找不到主题，只为每个页面输出 `found no layout file for "html" for kind "page"` 之类的信息，让人以为是模板坏了。本项目真的被这一段咬过一次——站点渲染出来完全没有主题，而所有报错都指向模板。

规避方法只有一条，但它必须成为习惯：**所有裸键写在第一个 `[table]` 之前**。这个仓库在 `hugo.toml` 和 `params.toml` 的顶部各放了一段注释说明这一点，就是为了让下一个人先看见它。如果你正在给某个表加键，就把键写在表里并缩进，别写在文件的末尾。

## 用 hugo config 定位生效值

配置有形状以后，剩下的工作是确认你改的那一层真的生效了。别猜，直接问构建工具：

```bash
hugo config
```

它打印的是合并主题、`_default`、环境三层，再叠加 Hugo 内置默认值之后的完整配置。输出很长，通常配合过滤使用：

```bash
hugo config | grep -E 'theme|pagerSize|sidebarSections'
```

在 PowerShell 下等价写法是 `hugo config | Select-String 'theme|pagerSize|sidebarSections'`。如果只想看模块挂载，`hugo config mounts` 单独打印 `[module]` 的结果——当模板或资源突然"消失"时，先看这个，它能立刻告诉你某个挂载点还在不在。

{{< note >}}
`[module]` 下的 `[[module.mounts]]` 值得单独记一笔：只要声明了任意一个挂载，Hugo 的默认挂载就会被整体替换。这个站点因此把 `content`、`static`、`assets`、`layouts`、`i18n`、`data`、`archetypes` 七个都写全了。少写一个，那个组件就会安静地不再参与构建。
{{< /note >}}

配置改完的标准动作不是刷新浏览器，而是跑一次构建并读输出：

```bash
hugo --logLevel warn
```

警告往往比错误更能说明问题。想让第一处警告直接变成失败，加上 `--panicOnWarning`，这样一次不小心的改动不会悄悄留在构建日志里。

## 参考

{{< docref "configuration/"  >}}
