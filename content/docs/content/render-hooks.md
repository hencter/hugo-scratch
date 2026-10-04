+++
title = '渲染钩子'
linkTitle = '渲染钩子'
description = '七个渲染钩子各自接管一种 Markdown 元素的输出，这一页把每一个都跑一遍，并说明它替 Markdown 补上了什么。'
date = 2026-02-22
weight = 50
difficulty = 'advanced'
estimatedTime = 25
prerequisites = ['/docs/content/markdown/']
outcomes = ['知道七个渲染钩子的输入与输出', '能给站内链接和图片加上构建期校验', '知道 Markdown 记法短代码的标题为什么能进目录']
tags = ['render hooks', '模板']
+++

渲染钩子是 Markdown 与最终 HTML 之间的那一层：**某一种元素**在渲染时改走你的模板。它们必须是「一种元素一个文件」，文件名固定，位置固定：

{{< filetree >}}
themes/hugo-scratch-theme/layouts/_markup/
├── render-heading.html        h1–h6
├── render-image.html          ![alt](src "title")
├── render-link.html           [text](href "title")
├── render-codeblock.html      围栏代码块
├── render-blockquote.html     引用，含 > [!NOTE]
├── render-table.html          表格
└── render-passthrough.html    $$…$$ / \(…\) 数学
{{< /filetree >}}

**目录名是 `_markup`**，不是 `_partials`，也不是 `partials`。写错位置的模板不会报错，它只是永远不被调用——页面看起来正常，只是少了那层加工。

## 标题：id、属性与锚点

钩子拿到 `.Level`、`.Text`、`.Anchor`、`.Attributes` 和 `.PlainText`。它做三件事：保留 Hugo 算出的 id（除非正文用 `{#custom}` 指定了别的）、把块属性透传到元素上、补一个悬停才出现的锚点链接。

### 这个小节的 id 是自定义的 {#hook-heading-id}

写法是 `### 这个小节的 id 是自定义的 {#hook-heading-id}`。右侧目录里的链接指向的也是这个 id——因为目录和钩子读的是同一份 `.Fragments`。

## 图片：宽高、懒加载与缺图告警

钩子调用 `resolve-image.html`，依次在**页面包**、`assets/`、`static/` 里找图。找到就输出 Hugo 读出的固有宽高——这是页面加载时不跳动的根本原因：

![默认分享卡](/images/og-default.png)

Markdown 的 title 会变成 `<figcaption>`：

![默认分享卡](/images/og-default.png "写 title 就会多出一个图注")

远程图片不受构建期校验约束（离线无法验证），但站内路径找不到文件时，钩子会 `warnf`：

> [!WARNING]
> 配合 `--panicOnWarning`，一张找不到的图片会让构建失败。这条规则的价值在于：坏图片在预览里只是空白，上线后才变成读者看到的破图。

## 链接：站内的校验、站外的 rel

站内相对链接会被解析一次，解析不到就告警——页面改名之后，所有指向它的链接会在**下一次构建**就暴露出来，而不是等读者点到 404：

- 站内：[Markdown 与扩展语法](/docs/content/markdown/)
- 站外：[Hugo 官方文档](https://gohugo.io/documentation/)

外链会被加上 `rel="noopener"` 与一个类名，箭头由 CSS 补上——因为钩子只负责让「这是外链」这件事在 HTML 里成立，箭头是样式问题。

校验可以用配置关掉（`validateInternalLinks = false`），用于那些指向部署后才存在的路径的站点。

## 代码块：高亮、语言标签与复制按钮

钩子里的核心是一行：

```go-html-template
{{ $result := transform.HighlightCodeBlock . }}
```

它返回高亮后的 HTML，钩子再自己包一层 `div.highlight`，补上语言标签、可选的文件名，以及一个复制按钮。复制按钮是**服务端渲染**的，所以即使 JS 打包失败它也在页面上，只是不工作：

```toml {filename="config/_default/hugo.toml"}
[markup.highlight]
  noClasses = false
```

`noClasses = false` 是这一整套能跟随明暗主题的前提：Chroma 输出类名而不是内联颜色，颜色才可能由两张样式表分别定义。

## 引用：五个提示词映射到四种样式

钩子先看 `.AlertType`。有值就转成提示块——**和 `{{</* note */>}}` 短代码共用同一个 partial**，所以正文里的提示块和短代码写的提示块不可能长成两个样子；没有值就原样输出引用：

> 文档里最贵的部分，往往是没人写下来的那些坑。

> [!IMPORTANT]
> GitHub 的提示词有五个：NOTE / TIP / IMPORTANT / WARNING / CAUTION，而样式只有四种。映射表写在钩子里，因为 `T "important"` 会是一条缺翻译——这正是「不要在模板里把任意字符串直接丢给 `T`」的一个具体例子。

## 表格：包一层，让横向滚动发生在表格里

钩子把表格包进 `.table-wrap`。这一层看起来多余，直到你在手机上打开一个七列表格：没有它，撑宽的是整个页面，正文、页脚、导航会一起跟着横移。

| 钩子 | 上下文里的关键字段 | 本页是否已触发 |
| --- | --- | :---: |
| `render-heading` | `Level`、`Text`、`Anchor`、`Attributes` | 是 |
| `render-image` | `Destination`、`Text`、`Title`、`Page` | 是 |
| `render-link` | `Destination`、`Text`、`Title`、`Page` | 是 |
| `render-codeblock` | `Type`、`Inner`、`Options` | 是 |
| `render-blockquote` | `Text`、`AlertType`、`AlertTitle` | 是 |
| `render-table` | `THead`、`TBody`、`Attributes` | 是 |
| `render-passthrough` | `Inner`、`Type` | 见下 |

{.no-wrap-first-col}

## 数学：包一层，交给前端的渲染器

`passthrough` 扩展把定界符原样交出来，钩子只负责把它们包进一个稳定的容器：

$$ e^{i\pi} + 1 = 0 $$

行内同样处理：\(\nabla \cdot \vec{E} = \rho / \varepsilon_0\)。主题**不加载**渲染器；没有渲染器时读者看到源码，这比看着像坏掉的标记要诚实，而且让从不写公式的站点不必付那份脚本钱。

## 一个容易被忽略的后果

Markdown 记法的短代码（`{{%/* tabs */%}}`、`{{%/* steps */%}}`）之所以能让内部标题进入目录，是因为它们跑在 Markdown 渲染器**之前**，其输出会被重新解析成 Markdown 块。标准记法的短代码跑在之后，`.Inner` 是未渲染的文本，标题就永远不会出现在 `.Fragments` 里。

这条差别直接影响选型：**内容里有标题、且希望它被目录收录，就只能用 Markdown 记法。**[^1]

[^1]: 上游文档：[Render hooks](https://hugozh.cn/render-hooks/)
