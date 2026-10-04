+++
title = '配置'
linkTitle = '配置'
description = '配置放在哪里、按什么顺序合并，以及为什么这份配置会被 TOML 的位置规则咬一口。'
date = 2026-01-14
weight = 20
difficulty = 'beginner'
estimatedTime = 15
prerequisites = ['/docs/start/directory-structure/']
outcomes = ['说清主题、_default、环境配置三层的优先级', '用 hugo config 查出一个生效值的出处', '知道裸键写在 [table] 下面会发生什么']
tags = ['Hugo', '配置']
+++

Hugo 的配置看起来最简单：写一个 `hugo.toml`，键值对放进去就行。但这个站点有主题要叠加、有生产环境要特判、还有好几个互不相干的关注点，塞进一个文件以后，改一个参数要在几百行里找它属于哪个表。所以配置被拆开了——拆开之后顺带得到一条明确的优先级链，以及一个必须遵守的位置规则。

## 为什么是 config/_default 而不是根目录的 hugo.toml

Hugo 支持两种布局：根目录一个 `hugo.toml`，或者一个 `config/` 目录，里面每个子目录是一层配置。这个站点选了后者，原因是它能同时表达三件根目录单文件表达不了的事：

- **关注点分离。** 主配置在 `config/_default/hugo.toml`，语言与地区设置在同目录的 `languages.toml`，模板读取的参数在 `params.toml`，导航在 `menus.zh-cn.toml` 与 `menus.en.toml`。找"日期格式在哪"不需要通读全文。
- **环境差异有地方放。** 生产构建的额外设置写进 `config/production/hugo.toml`，本地开发完全不受影响，也不需要在命令行上追加参数。
- **主题与站点的边界是文件级的事实。** 主题自己那份 `hugo.toml` 是链条的第一环，站点的配置是第二环，谁覆盖谁一眼可见。

代价是"配置在哪"这个问题不再有唯一答案，所以你需要在脑子里保留下面那条合并顺序。

## 三份配置，一个生效结果

构建时，Hugo 按固定的顺序把配置一层层叠起来，后面的覆盖前面的：

1. **主题的 `hugo.toml`** ——`themes/hugo-scratch-theme/hugo.toml`。它只能提供 `params`、`menu`、`outputformats` 和 `mediatypes` 这几类内容；文件里其它顶层键（比如 `baseURL`）会被忽略。这里放的是"没有站点配置也能跑起来"的默认值。
2. **`config/_default/`** ——站点的配置本体，优先级高于主题，也就是说 `config/_default/params.toml` 里的每个键都压过主题的同名键。
3. **`config/<环境>/`** ——环境层，优先级最高。一次普通的 `hugo` 构建使用生产环境，因此读取 `config/production/hugo.toml`；如果设置了别的环境名，Hugo 会去找同名目录，找不到就不叠加。

同一个目录里的多个文件是平级的，不构成嵌套层次：`hugo.toml`、`params.toml`、`languages.toml` 谁都不会覆盖谁，它们各自负责不同的树。

{{< note >}}
本项目没有在根目录保留 `hugo.toml`，这不是疏忽。根目录单文件与 `config/` 目录同时存在时，两者都会被读取，谁最后生效取决于读取顺序——要避免这种难以排查的局面，只留一种布局。
{{< /note >}}

## 三个文件，各管一摊

- **`config/_default/hugo.toml`** ——站点身份与构建行为：`baseURL`、`title`、`locale`、`defaultContentLanguage`、`enableGitInfo`、`timeZone`、`theme`、`[module]` 挂载、`[frontmatter]` 日期来源、`[markup]`、`[taxonomies]`、`[pagination]`、`[outputs]`、`[cascade]` 等等。要改站点的运行方式，从这里开始。
- **`config/_default/languages.toml`** ——`zh-cn` 和 `en` 两个语言的 `label`、`locale`、`weight`、`title`，以及各自 `[<lang>.params]` 下的 `dateFormat`。它还记录了内容配对规则：`page.md` 是默认语言（`zh-cn`），`page.en.md` 是英文版，Hugo 按同一目录下的同名基准配对，所以不需要把内容拆成两棵目录树。
- **`config/_default/params.toml`** ——模板读取的站点级参数，例如 `description`、`author`、`repoURL`、`[ui]` 下的一批显示开关、`[nav] sidebarSections`、`[seo]` 等。主题为每个键准备了默认值，所以删掉一项不会报错，只会退回主题的值。

菜单不在这里，而在同目录的 `menus.zh-cn.toml` 与 `menus.en.toml`——理由见[导航](/docs/configuration/navigation/)。

## TOML 的位置陷阱

TOML 里，一个裸键属于它上方最近的那个表头。写成这样：

```toml
[frontmatter]
  lastmod = [':git', 'lastmod', 'date']

theme = ['hugo-scratch-theme']
```

`theme` 被写在 `[frontmatter]` 下面，于是它真正的名字是 `frontmatter.theme`，而不是顶层 `theme`。Hugo 不会因此报配置错误——它只是找不到主题，然后为每个页面抱怨找不到布局文件。这个坑本项目真实踩过一次：站点渲染出来完全没有主题，而报错全部指向模板。

判断规则很简单：**所有裸键写在第一个表头之前**。本项目在 `hugo.toml` 和 `params.toml` 的顶部都用注释写明了这一点，就是为了让下一个人先看见它。如果你需要给某个表加键，就把它写进那个表里，并且用两格缩进显示从属关系。

{{< warning title="改完配置后先看构建输出" >}}
配置错误经常表现为"页面突然没样式"或"某个参数不生效"，而不是一条清晰的报错。改完 `config/` 下的文件之后，跑一次构建并读它的输出，比在浏览器里刷新页面更快定位问题。
{{< /warning >}}

## 用 hugo config 看生效值

别靠猜。`hugo config` 打印本次构建真正使用的完整配置，包含主题、`_default`、环境三层合并之后的结果，以及 Hugo 自己的内置默认值：

```bash
hugo config
```

输出很长，配合管道过滤：

```bash
hugo config | grep -E 'theme|pagerSize|sidebarSections'
```

这是一条通用规律：任何"这个值到底是多少"的问题，答案都应该来自构建工具，而不是来自记忆或文档。同理，Hugo 的完整命令行参考可以用 `hugo gen doc --dir` 生成本地文档，那是一份与已装版本严格对应的说明。

{{< tip >}}
`hugo config --printZero` 会连零值一起打印，方便确认某个键确实没有被设置，而不是恰好等于默认值。
{{< /tip >}}
