+++
title = '隐私'
linkTitle = '隐私'
description = '这个站点在浏览器里请求什么、不请求什么，以及什么时候才会有第三方请求。'
date = 2026-01-05
weight = 10
+++

## 结论

**页面只请求同源的资源。** 样式表、脚本、图片和 `search.json` 都来自本站，
`https://scratch.hugozh.cn/` 之外没有任何请求。
默认配置下，这个站点不设置任何 Cookie，也不向任何分析服务报告访问。
下面逐条说明，并给出可以自己验证的做法。

## 页面实际请求了什么

| 资源 | 来源 | 说明 |
| --- | --- | --- |
| `/css/bundle.min.<hash>.css` | 本站 | 主题 `main.css` 与站点 `custom.css` 合并后的样式表 |
| `/js/main.<hash>.js` | 本站 | `js.Build` 打出的一个 bundle |
| 页面里的图片 | 本站 | 通过页面资源、`assets/` 或 `static/` 解析 |
| `/search.json` | 本站 | **只在读者打开搜索框时**才抓取 |
| 图标与 webmanifest | 本站 | `static/` 下的 favicon 与站点清单 |

两个资源名里都带内容哈希，并配 `integrity` 属性：哈希对不上，浏览器会拒绝执行，
这是**完整性**保证，不是追踪手段。

## 统计默认是关的

`layouts/_partials/analytics.html` 做两件事：先看 `[params.analytics]` 有没有值，
再看这次构建是不是生产构建。两个条件都满足才会输出标签。

仓库里的值现在是空的：

```toml
[analytics]
  googleAnalytics = ''
  plausibleDomain = ''
```

所以现在的站点没有任何分析脚本可用，也没有任何东西会去联系 Google 或 Plausible。
这份配置同时还在 `[privacy.googleAnalytics]` 里写了 `disable = true`。

把值填上之后，情况会变：Google Analytics 的 gtag 会从 `googletagmanager.com`
加载，Plausible 会从 `plausible.io` 加载——**只在生产构建里**。
`hugo server` 期间的访问永远不会进入统计，代价是本地预览也看不到数据。

## 三种常见的第三方请求，这里都没有

**没有字体 CDN。** 字体是系统字体栈，写死在
`assets/css/tokens.css` 的 `--font-sans` 与 `--font-mono` 里
（`-apple-system`、`Segoe UI`、`ui-monospace` 之类），
所以字体不会产生任何网络请求，页面也不会因为等字体而闪一下。

**没有 YouTube 嵌入。** 主题重写了 Hugo 内置的 `youtube` 短代码：
它渲染的是一个指向 `youtube-nocookie.com` 的**链接**，而不是 `<iframe>` 播放器。
内置版本会在页面打开时就联系 YouTube 并写入 Cookie；这里要等读者自己点出去，
请求才发生——而那时读者已经离开了本站。

**没有评论插件。** `layouts/_partials/comments.html` 渲染的是一个指向仓库 issue 的链接，
指向 `github.com`，而链接在读者点击之前不产生请求。

## 搜索索引是按需抓取的

搜索框索引的是一份 JSON：`/search.json`（英文是 `/en/search.json`）。
它由首页的 `search` 输出格式在构建时生成，但**不会随页面一起加载**。

`assets/js/modules/search.js` 里的 `load(params)` 只在读者第一次打开搜索对话框时调用
`fetch`，并且带 `credentials: 'same-origin'`。抓到之后索引留在内存里，
本次浏览期间再搜索不会重复请求。不用搜索的读者，一个字节的索引也不会下载。

## 自己怎么验证

1. 打开任意页面，按 {{< kbd "F12" >}} 打开开发者工具的 Network 面板，刷新。
   列表里的请求域名应当只有本站。
2. 打开搜索框（{{< kbd "Ctrl" >}} + {{< kbd "K" >}}），这时才会多出一个
   `search.json` 请求。
3. 查看 Application / Storage：本站不写入 Cookie。主题选择等本地状态只存在这个浏览器里。
4. 换一个干净的无痕窗口再打开一次，请求列表应该和第一次完全一样——
   没有多出来的第三方域名，也没有被缓存的脚本替你解释这一切。

如果你在自己的机器上看到别的域名，那多半是浏览器扩展或代理加进来的，不是这个站点发出的。
