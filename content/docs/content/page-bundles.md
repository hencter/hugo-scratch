+++
title = '页面包（page bundles）'
linkTitle = '页面包'
description = '叶子包与分支包的区别、页面资源怎么被找到，以及图片为什么先问包再问 assets/。'
date = 2026-02-20
weight = 30
difficulty = 'intermediate'
estimatedTime = 20
prerequisites = ['/docs/content/', '/docs/content/front-matter/']
outcomes = ['判断一个目录该用 index.md 还是 _index.md', '把图片放进包并用 Markdown 引用它', '看懂 resolve-image.html 的三级查找顺序']
tags = ['Hugo']
+++

在 Hugo 里，目录不只是一个放文件的抽屉。带 `index.md` 的目录会变成一页，目录里的其他文件变成这一页的资源；带 `_index.md` 的目录会变成一个章节，目录里的 `.md` 各自成为页面。这一页讲清这两者的差别，以及主题是怎么去找一张图片的。

## 三种包

| | 叶子包（leaf bundle） | 分支包（branch bundle） |
| --- | --- | --- |
| 索引文件 | `index.md` | `_index.md` |
| 页面类型 | `page` | `home`、`section`、`taxonomy`、`term` |
| 下级页面 | 不允许 | 可以有 |
| 资源 | `.Resources` 取到 | `.Resources` 取到，但排除下级包 |

第三条是关键约束：**叶子包不能套叶子包**。`content/blog/hugo-pipes/` 里再建一个 `content/blog/hugo-pipes/part-two/index.md`，第二个不会被当成子页面，只会变成一个普通文件资源。

第三种是 headless 包：内容与资源都在，但不作为独立页面发布，只留给别的页面取用。它由 front matter 里的 `headless` 加上 `build` 选项定义：

```toml
+++
title = '图片素材库'
headless = true
+++

[build]
  publishResources = true
```

`headless = true` 让这一页不产出 HTML，也从列表、RSS 与站点地图里消失，但 `.Resources` 仍然可以被别的模板取到——把它当作一个只供引用的素材目录。`build.publishResources` 决定包里的文件要不要真的复制进 `public/`。键名是 `build`，不是 `_build`。

## 一个真实的叶子包

`content/blog/hugo-pipes/` 就是这个站点里的叶子包，目录里只有两样东西：

{{< filetree >}}
content/
└── blog/
    ├── _index.md
    ├── hello/
    │   └── index.md
    └── hugo-pipes/
        ├── index.md
        └── cover.png
{{< /filetree >}}

`index.md` 决定 URL 是 `/blog/hugo-pipes/`，`cover.png` 与它同级，因此是这一页的资源，可以用相对路径直接引用：

```markdown
![Hugo Pipes 的构建产物](cover.png)
```

注意是 `cover.png`，不是 `/cover.png`，也不是 `/images/cover.png`。相对路径让 Hugo 知道先在这一页的包里找；写成 `/images/...` 就变成站点根路径，只有 `assets/` 或 `static/` 里有对应文件时才能解析。

## 主题怎么找一张图片

`layouts/_partials/resolve-image.html` 是唯一处理这件事的地方，Markdown 图片（`_markup/render-image.html`）、`figure` 短代码和 Open Graph 标签都调它。它按固定顺序问三个问题：

1. `$page.Resources.GetMatch $clean`——先问当前页的包；
2. `resources.Get $clean`——再问站点的 `assets/`；
3. 都没有，就当它在 `static/` 里，用 `absURL` 拼出地址。

从包开始问，是因为只有包里的资源和 `assets/` 里的资源能拿到 `MediaType` 和宽高：`.Width`、`.Height` 让模板能写出 `width`/`height` 属性，图片加载时页面不会跳。`static/` 里的文件是原样复制的，Hugo 不知道它多大，所以那一支返回的 `width` 和 `height` 都是 0。

这个顺序也解释了为什么同名文件放在包里会「赢过」`assets/`：包更靠近这一页，覆盖站点级默认值是它的用途。`resolve-image.html` 返回的是 `{url, width, height, resource}` 四个值，其中 `resource` 为 `false` 就说明这一支走的是 `static/`。

{{< warning "文件名要唯一到能匹配" >}}
`Resources.GetMatch` 接受 glob，`cover.png` 能匹配 `cover.png`，`*.png` 会匹配包里的**第一张** PNG。包里有多张图时别用太宽的 glob，否则换一次文件顺序就换一张图。
{{< /warning >}}

## 分支包的资源与 cascade

分支包也能有资源，但它的 `.Resources` **不包含下级包里的文件**。`layouts/_partials/head/schema.html` 拿站点的 `params.images` 去调 `resolve-image`，用的就是首页这个分支包的上下文——`images/logo.svg` 是 `assets/` 里的资源，因此走的是第二条查找路径。

分支包另一个用途是给一整片目录统一设置值，靠站点级的 `cascade`。`config/_default/hugo.toml` 里有两条，其中一条把 `/legal/**` 下的所有页面的 sitemap 频率改成 `yearly`：

```toml
[[cascade]]
  [cascade.sitemap]
    changefreq = 'yearly'
    priority = 0.2
  [cascade.target]
    path = '/legal/**'
```

`cascade.target.path` 用 glob 选页面，剩下的键合并进它们的 front matter。想给某一章的所有页面加上同一个 `tags` 或 `notice`，写在这里比在每页重复一遍可靠——重复的那份早晚会漏掉一页。

## 什么时候该拆包

页面只有一张配图时，`content/blog/hello/index.md` 与 `cover.png` 同级就够了。图片要在多个页面复用、或者需要图片处理（缩放、裁剪、转格式）时，把它放进 `assets/`：`assets/` 里的资源**只在被引用时才发布**，而 `static/` 里的一切无论用不用都会复制进 `public/`。这条差别在图片多的站点上就是最终产物体积的差别。

## 参考

{{< docref "content-management/page-bundles/"  >}}
