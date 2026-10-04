+++
title = '速查'
linkTitle = '速查'
description = '把前面各章压缩成可查的表：特性对应到文件、配置键与模板，读完文档之后从这里回查。'
weight = 90
+++

前面几章是按阅读顺序写的，这一章是反过来用的：知道要改什么，回查它在哪。所以它不重复叙述，只给指向具体文件和配置键的入口。

## 这个仓库里的文件都能自己说话

本主题几乎所有配置键的默认值都写在文件顶部的注释里，键名、默认值与出处三者在一起，比任何速查表都准：

- `config/_default/hugo.toml` —— `[outputs]`、`[frontmatter]`、`[markup]`、`[taxonomies]`、`[cascade]`、`[imaging]`、`[caches]`、`[security]`、`[privacy]`；
- `config/_default/params.toml` —— 界面开关、社交图片、验证码、评论仓库；
- `config/_default/languages.toml` —— 每个语言站点的 `label`、`locale` 与日期格式；
- `themes/hugo-scratch-theme/hugo.toml` —— 主题能声明的四类键：`params`、`menu`、`outputformats`、`mediatypes`；
- `data/changelog.toml` —— 更新日志的唯一数据源。

想确认一个键在当前 Hugo 里到底存不存在、默认值是多少，不要猜，也不要照抄网页：跑一次 `hugo config`，它打印的是**合并之后**的实际配置。

## 从「我要改什么」找文件

| 想做的事 | 去哪儿 |
| --- | --- |
| 加一页文档 | `content/docs/` 下新建 `.md`，写 `title` 与 `weight`，见 [front matter](/docs/content/front-matter/) |
| 给一页配图 | 做成叶子包，图片与 `index.md` 同级，见 [页面包](/docs/content/page-bundles/) |
| 改侧栏范围 | `config/_default/params.toml` 的 `[params.nav] sidebarSections` |
| 改导航菜单 | `config/_default/menus.<lang>.toml`，`pageRef` 关联页面 |
| 改样式 | 站点自己的 `assets/css/custom.css`，不要覆盖主题的 `main.css` |
| 改界面文案 | `themes/hugo-scratch-theme/i18n/<lang>.toml`，中英两份都要加 |
| 增删机器可读输出 | `config/_default/hugo.toml` 的 `[outputs]`，模板名见 [输出格式](/docs/templates/output-formats/) |
| 改站点地图频率 | 页面的 `[sitemap]` 表，或站点级 `[[cascade]]` |
| 换评论系统 | 在站点里新建 `layouts/_partials/comments.html` 覆盖主题版本 |

## 三个必须记住的边界

{{< warning >}}
`.Site.Data`、`.Page.IsNode`、`site.Sites`、`.Language.LanguageName`、`cascade._target`、`_build` 都已经废弃。替代写法分别是 `hugo.Data`、`.IsPage` / `.IsBranch`、`hugo.Sites`、`.Language.Label`、`cascade.target`、`build`。在 `--panicOnWarning` 下，一条弃用告警就是一次构建失败。
{{< /warning >}}

另外两条不那么显眼，但同样会让人查半天：

- 内容里的短代码分隔符必须转义，代码块里也一样：

  ```text
  {{</* note */>}} 内容 {{</* /note */>}}
  {{%/* tabs */%}} … {{%/* /tabs */%}}
  ```
- 模板里 `and`、`or` 会求值每一个参数，`.Date` 是结构体永远为真，`default true` 表达不了显式的 `false`。这三件事在 [模板与查找顺序](/docs/templates/templates/) 里都有能跑的例子。

## 用构建命令代替记忆

```bash
hugo version      # 版本决定哪些键与命令存在
hugo config       # 合并后的实际配置，含默认值
hugo list all     # Hugo 眼里的内容清单
hugo --ignoreCache --panicOnWarning   # 一条告警即失败
```

四条命令覆盖了「我改的对不对」的大部分问题。剩下的判断只有一条：产物里有没有那一页——`public/` 才是真相，控制台不报错不等于页面渲染了。

要按「哪个特性落在哪个文件」横向对照，看 [特性对照表](/docs/reference/feature-matrix/)；它把配置键、模板、短代码和输出格式按特性排成一张表，适合改动前先确认影响面。
