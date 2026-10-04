+++
title = '模板'
linkTitle = '模板'
description = '这个主题的模板系统：baseof 定义契约，页面类型只提供 main，输出格式决定同一份内容还能长成什么样子。'
weight = 40
+++

这一章讲模板，但只讲这个主题真实使用的写法：`layouts/baseof.html` 定义文档契约，`layouts/page.html`、`layouts/section.html` 之类只填一个 `main` 块；输出格式把同一份内容再产出一遍，变成 Markdown 和 JSON。没有 `layouts/_default/`，也没有 `single.html`。

## 三页的分工

- [模板与查找顺序](/docs/templates/templates/) 从 `baseof.html` 一路走到 `page.html`，说明每一层读到什么、`{{ define "main" }}` 之外还有哪些块可以覆盖，以及一个 partial 怎么返回值而不打印 HTML；
- [输出格式](/docs/templates/output-formats/) 解释为什么同一个首页会同时存在 `index.html`、`index.md`、`llms.txt`、`search.json` 和 `pages.json`，以及谁来命名这些文件。

两页合起来覆盖一次请求从内容到字节的全过程。

## 一个目录，三种命名习惯

这个主题的模板目录只用三种名字：

| 目录 | 放什么 | 例子 |
| --- | --- | --- |
| `layouts/` | 页面类型模板与输出格式模板 | `page.html`、`home.llms.txt`、`list.md` |
| `layouts/_partials/` | 被调用的片段，可返回值 | `toc.html`、`layout/flags.html` |
| `layouts/_shortcodes/` | 短代码实现 | `note.html`、`tabs.html` |
| `layouts/_markup/` | 渲染钩子 | `render-image.html` |

下划线开头的目录是 Hugo 新模板系统的约定；站点自身的 `layouts/` 永远优先于主题。要改一个 partial，不必动子模块里的主题——在仓库根的 `layouts/_partials/` 放一个同名文件即可覆盖。

## 三条容易踩的规则

{{< warning >}}
Go 模板的 `and` / `or` 是函数，**每个参数都会被求值**。`and $p $p.Title` 在 `$p` 为 nil 时照样 panic，因为 `$p.Title` 已经被求值了。要短路就用嵌套的 `with`，或先算出一个安全的字符串。
{{< /warning >}}

- **`default true` 表达不了显式的 `false`**：`default` 只在值为空时接手，调用方写下的 `false` 会被当成「没有值」，于是被替换成 `true`。模板里的开关要用 `isset` 或直接比较值，`layouts/_partials/layout/flags.html` 就是这么处理 `showSidebar` 的：先 `index`，再用 `eq ... nil` 判断键是否存在。
- **`.` 是当前上下文，`$` 是模板最开始的那个上下文**。partial 里嵌套 `range` 之后想回到页面，写 `$`，不要靠猜。

## 谁在读这些模板

模板不是给人看的文档，页面上每个区块都对应一个文件：侧栏来自 `layouts/_partials/sidebar.html`，本页目录来自 `layouts/_partials/toc.html`，页脚的 llms.txt / pages.json 链接来自 `layouts/_partials/footer.html` 读取 `.OutputFormats`。改模板之前，先在这些文件里找到那一块——比从 CSS 类名反推快得多。
