+++
title = '短代码'
linkTitle = '短代码'
description = '主题自带的全部短代码，每一种都在这一页真实调用一次，并说明它用哪种记法、为什么。'
date = 2026-02-18
weight = 40
difficulty = 'intermediate'
estimatedTime = 20
prerequisites = ['/docs/content/markdown/']
outcomes = ['知道每个短代码的参数与记法', '能判断一个新组件该用标准记法还是 Markdown 记法', '知道怎么在站点里覆盖主题的短代码']
tags = ['短代码', '模板']
+++

短代码写在 `layouts/_shortcodes/<名字>.html`。主题的短代码和站点的短代码在同一个查找链里，**站点里的同名文件优先**——这就是覆盖主题某个短代码而不用改主题的办法。[^1]

## 两种记法，决定的不是外观而是渲染顺序

| 记法 | 写法 | `.Inner` 是什么 | 里面的标题进目录吗 |
| --- | --- | --- | --- |
| 标准记法 | `{{</* name */>}}` | 未渲染的原文，需要自己 `markdownify` | 不进 |
| Markdown 记法 | `{{%/* name */%}}` | 已经过 Markdown 渲染的 HTML | 进 |

这一页对两种都有真实调用。判断标准只有一条：**内容里如果会出现标题，而且希望它出现在右侧目录里，就必须用 Markdown 记法。**

## 提示块：note / tip / warning / danger

四个名字共用 `layouts/_partials/shortcodes/callout.html`，所以它们的外观不可能各走各的。标准记法，内容会被 `markdownify`：

{{< note >}}
`note` 是最中性的一个。可以用 `title` 命名参数换成自定义标题。
{{< /note >}}

{{< tip "先做这一步" >}}
这个框的标题用的是**位置参数**。写成命名参数 `title` 效果完全相同，但一次调用里不能混用两种。
{{< /tip >}}

{{< warning >}}
标准记法里的 `.Inner` 是**未渲染**的原文，所以主题在 partial 里调用了 `markdownify`。反过来说，标题不会进目录。
{{< /warning >}}

{{< danger >}}
这个框的用途是「按下去会坏东西」。颜色和图标来自 `--color-danger`，跟着明暗主题走。
{{< /danger >}}

## 折叠块：details

原生 `<details>`，不需要 JavaScript：

{{< details title="展开看一段说明" >}}
浏览器已经实现了折叠语义，短代码只是补上版式和文案。`open="true"` 可以让它默认展开——注意这里必须按**字符串**比较，因为 `open=false` 在模板里是真值。
{{< /details >}}

## 行内小件：badge 与 kbd

不用闭合，参数即全部内容：{{< badge "New" >}}、{{< badge text="Beta" tone="accent" >}}、按键 {{< kbd "Ctrl" >}} + {{< kbd "K" >}} 打开搜索框。

注意第二个 badge：短代码**不能混用位置参数与命名参数**（写成 `badge "Beta" tone="accent"` 会直接报 "cannot mix named and positional parameters"），要么全位置，要么全命名。

## 构建环境的事实：version

{{< version >}} —— 短代码读的是 `hugo.Version`。内容里写「本项目需要某个版本以上」会过期，读构建时的实际版本不会。

## 目录树：filetree

标准记法，内容**不会**经过 Markdown——缩进、`*` 和 `-` 正是目录树的本体：

{{< filetree >}}
content/
├── _index.md              首页（zh-cn 是默认语言）
├── _index.en.md           英文首页
├── docs/
│   ├── _index.md
│   └── content/
│       ├── _index.md
│       └── shortcodes.md  ← 就是这一页
└── changelog/
    ├── _index.md
    └── _content.gotmpl    内容适配器：从 data 生成页面
{{< /filetree >}}

## 标签页：tabs + tab（Markdown 记法）

父子两个短代码共用一个 `.Store`。子级渲染时把自己的标题追加进**父级**的 store，而嵌套短代码是**由内向外**渲染的，所以父级执行时两张标签都已经登记好了——这就是按钮能服务端渲染出来的原因。

{{% tabs %}}
{{% tab "第一种写法" %}}
内容先经过 Markdown 渲染，所以这里可以写 **加粗**、列表和代码：

- 记法是 `{{%/* tabs */%}}`
- 子级是 `{{%/* tab "标题" */%}}`
{{% /tab %}}
{{% tab "第二种写法" %}}
父级还可以拆掉一层，例如把整组放在别处再引用；控制器只依赖面板上的 `role="tabpanel"`。
{{% /tab %}}
{{% /tabs %}}

## 步骤：steps + step（Markdown 记法）

输出是真正的 `<ol>`：读屏软件会念「共 3 项」，打印时序号也在。

{{% steps %}}
{{% step "装依赖" %}}
在站点根目录执行 `npm ci`——Tailwind 的 CLI 从这里装。
{{% /step %}}
{{% step "确认统计文件" %}}
`hugo_stats.json` 由 `[build.buildStats] enable = true` 产出，Tailwind 靠它知道哪些类真的用上了。
{{% /step %}}
{{% step "构建" %}}
`hugo --ignoreCache`，然后读 `public/` 里的产物，而不是重读源文件。
{{% /step %}}
{{% /steps %}}

## 分栏：columns + column（Markdown 记法）

列数通过自定义属性传给 CSS，窄屏折叠成单列由样式决定，不由模板决定：

{{% columns cols=2 %}}
{{% column %}}
**左栏。** 内容先渲染成 Markdown，再交给短代码排布。
{{% /column %}}
{{% column %}}
**右栏。** `cols` 参数只影响 `--columns` 这个自定义属性。
{{% /column %}}
{{% /columns %}}

## 图片：figure（覆盖内置）

站点的短代码优先于内置的同名短代码，所以 `layouts/_shortcodes/figure.html` 直接接管了它。它和渲染钩子、Open Graph 标签共用同一个 `resolve-image.html`，因此页面包里的图片在三处都能找到：

{{< figure src="/images/og-default.png" alt="社交分享卡的默认图" caption="宽高由 Hugo 从文件里读出来，写进 width/height，避免加载时页面跳动。" >}}

## 视频：video

短代码先检查文件是否存在，找不到就不吐一个坏掉的播放器，而是说清楚缺什么：

{{< video src="/videos/example.mp4" poster="/images/og-default.png" caption="把 example.mp4 放进 static/videos/ 之后，这里会变成一个可播放的 video 元素。" >}}

## 外部视频：youtube（覆盖内置）

内置版本会在页面加载时就插入 `<iframe>`，在读者开口之前就联系 YouTube 并写 Cookie。这里的覆盖版只渲染一个链接：

{{< youtube id="dQw4w9WgXcQ" title="示例：一个外部视频链接" >}}

主机名跟随 `[privacy.youtube] privacyEnhanced`，所以 Hugo 文档里那个配置键在这里仍然是有效的。

## 图表：mermaid

关闭时**可见地**退回一段高亮代码块，而不是吐一个没有渲染器的空容器：

{{< mermaid >}}
flowchart LR
  A[content] --> B(hugo)
  B --> C{hugo_stats.json}
  C --> D[Tailwind CLI]
  D --> E[public/css]
{{< /mermaid >}}

想真的渲染，就把 `[params.mermaid] enabled` 打开，并自行加载 mermaid 运行时——那是一份体积可观的第三方脚本，属于站点的决定，不属于主题的默认值。

## 目录：toc

打印 Hugo 为**本页**生成的 `.TableOfContents`。它和右侧那个目录是两套实现，正好可以对照：

{{< toc >}}

右侧目录由 `layouts/_partials/toc.html` 从 `.Fragments.Headings` 重建，因此能带 `aria-current` 和滚动高亮；`.TableOfContents` 返回的是一段现成的 `<nav>` 字符串，快，但标记不由你控制。两者收录的层级都由 `[markup.tableOfContents]` 的 `startLevel` / `endLevel` 决定，所以它们偶尔会不一致——那是配置，不是 bug。

## 更新日志：changelog

从 `data/changelog.toml` 直接渲染版本列表。同一个数据源还会被内容适配器展开成 `/changelog/` 下的一页页：

{{< changelog >}}

## 跨语言参考：docref

{{< docref "content-management/shortcodes/" >}}

它只有一件事要做：**同一次调用，在两种语言下指向各自的上游文档**。主机名来自 `config/_default/params.toml` 的 `[docs]`，所以中文页给出 hugozh.cn、英文页给出 gohugo.io，切换语言时参考文档跟着切换，而正文只写了一遍。链接上带 `hreflang` 与 `lang`，标明目标文档的语言；旁边印出主机名，因为"参考"在两个站点上不是一回事——一个是社区译文，一个是上游原文。

本站正文里的参考不这么写了：引用处用 Markdown 脚注（`[^1]`），定义放在页末，由 Goldmark 渲染成文末的脚注块。`docref` 因此保留给需要独立参考框的场合——它自带主机名与 `hreflang`，适合在一段正文里就地说明"去看上游哪一页"。

路径兼容性是量过的，不是猜的：抽样的 26 条文档路径在两个站点上都是 200。万一某天分叉，用 `zh=` 覆盖即可，调用形状不变：

```text
{{</* docref href="new/path/" zh="old/path/" */>}}
```

## 片段复用：include 与无头包

{{< include "build-gate" >}}

上面这一段不是写在这一页里的，而是 `{{</* include "build-gate" */>}}` 取来的。片段放在 `content/snippets/build-gate/index.md`，前置元数据里的 `headless = true` 让它成为一个**无头包**：Hugo 会渲染它的内容，但不发布任何东西——没有 URL、不进列表、没有 Markdown 孪生页。区别很实在：`public/snippets/` 在产物里根本不存在，而它的文字出现在两个引用它的页面上。

`site.GetPage` 能按路径找到无头包，这正是它能工作的原因；而 `.Pages`、`.RegularPages`、`.Sections` 这些页面集合里永远不会有它。

[^1]: 上游文档：[Shortcodes](https://hugozh.cn/content-management/shortcodes/)