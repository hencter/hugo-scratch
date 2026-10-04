+++
title = '用 Hugo Pipes 拼出 CSS 与 JS'
description = '两条管线的具体写法：css.Build 内联 @import、js.Build 打包并注入 @params，以及为什么只有 Tailwind 那一段需要 npm 装的 CLI。'
date = 2026-02-18
authors = ['Hencter Lew']
images = ['cover.png']
tags = ['Hugo', 'Hugo Pipes']
categories = ['工程实践']
series = ['从零搭一个 Hugo 站点']
+++

## 两条管线各自做了什么

![Cover of the Hugo Pipes post](cover.png "Processing with Hugo Pipes")

CSS 一侧：`assets/css/main.css` 里只有十条 `@import`，按顺序引入 `tokens.css`、
`chroma-light.css`、`chroma-dark.css`、`base.css`、`layout.css`、`components.css`、
`content.css`、`blocks.css` 和 `print.css`。`css.Build` 把它们内联成一个文件并重写其中的 `url()`；
随后 `resources.Concat` 把站点自己的 `assets/css/custom.css` 接在末尾，
最后由 `minify` + `fingerprint "sha384"` 改名。

JS 一侧：`assets/js/main.js` 用 `import` 拉进 `./modules/` 下的七个模块
（`theme.js`、`nav.js`、`search.js`、`toc.js`、`copy.js`、`tabs.js`、`back-to-top.js`），
`js.Build` 交给 esbuild 打成一个文件，输出 `format = 'iife'`、`target = 'es2020'`。

两边都只有一个产物、一次请求：管线把多个源文件折叠成一次传输。

## CSS 管线的完整写法

`layouts/_partials/head/css.html` 是本主题里唯一构建样式表的地方：

```go-html-template
{{- $parts := slice -}}
{{- with resources.Get "css/main.css" -}}
  {{- $parts = $parts | append (. | css.Build (dict "minify" false)) -}}
{{- else -}}
  {{- errorf "theme asset assets/css/main.css is missing — cannot build the stylesheet" -}}
{{- end -}}
{{- with resources.Get "css/custom.css" -}}
  {{- $parts = $parts | append (. | css.Build (dict "minify" false)) -}}
{{- end -}}

{{- $css := $parts | resources.Concat "css/bundle.css" -}}
{{- if hugo.IsDevelopment -}}
  <link rel="stylesheet" href="{{ $css.RelPermalink }}">
{{- else -}}
  {{- with $css | minify | fingerprint "sha384" -}}
    <link rel="stylesheet" href="{{ .RelPermalink }}" integrity="{{ .Data.Integrity }}" crossorigin="anonymous">
  {{- end -}}
{{- end -}}
```

四个细节值得单独指出：`css.Build` 的 `minify` 是 `false`，压缩交给后面的 `minify` 一次做完；
`resources.Concat "css/bundle.css"` 里的路径是**资源缓冲区名**，不是磁盘上的文件；
`errorf` 让缺失的入口文件变成构建失败，而不是一个没样式的页面；
开发构建输出 `css/bundle.css`，生产构建输出带哈希的 `css/bundle.min.<hash>.css`。

## JS 管线：`@params` 是虚拟模块

关键在传给 `js.Build` 的那个 `params` 值：

```go-html-template
{{- $params := dict
      "searchIndex" $searchIndex
      "i18n" (dict "copy" (T "copy") "searchNoResults" (T "searchNoResults")) -}}

{{- $opts := dict
      "minify"    (not $dev)
      "target"    "es2020"
      "format"    "iife"
      "sourceMap" (cond $dev "linked" "none")
      "params"    $params -}}
```

`params` 不是文件也不是环境变量：esbuild 会把它当成模块 `@params` 解析，
所以 `assets/js/main.js` 第一行 `import * as params from '@params';` 就能直接拿到这个字典。
这是整个站点里唯一一条从 Hugo 配置通往浏览器代码的路径——没有 JSON 端点，
没有往 `<body>` 上贴 `data-` 属性。搜索索引的 URL 和界面文案都从这里进浏览器，
也因此每种语言都有一份自己的 bundle 和自己的哈希。

## 哪些部分不需要 Node，哪些需要

不需要的部分占多数，因为 Hugo 自己做了：`css.Build` 用 Go 实现的 CSS 解析器处理 `@import` 与 `url()`，`js.Build` 内嵌 esbuild 做打包，`resources.Concat`、`minify`、`fingerprint` 都是模板函数。设计系统、脚本打包、压缩、加哈希、SRI，全都不需要 `package.json`。

需要 Node 的只有一处：Tailwind v4 那一段走官方集成 `css.TailwindCSS`，它调用的是 npm 装在站点根目录的 CLI。`package.json` 里因此只有两个包，CI 里也只有一步 `npm ci`。这个取舍是有意的——Tailwind 的按需生成要扫描渲染结果，那件事只有 Tailwind 自己的编译器做得准。

代价仍然清楚：没有 SCSS，没有 PostCSS 插件，也没有任意的 JS 转换器。换来的是除 Tailwind 之外，构建环境只有一个二进制。

## 文件级合并：主题资源和项目资源同名就换掉了

这是最容易在主题升级时吃到的坑。`assets/`、`layouts/`、`static/` 都是**文件级**合并：
同名文件后加入的层整个替换掉前一层，而不是逐行合并。
如果项目里放了一份 `assets/css/main.css`，它会**整个替换**主题的同名文件——
主题后续的每一条样式修正都不会再到达页面，而且构建不会给出任何提示。

正确的做法是用不同的文件名，靠顺序而不是替换来组合。这个站点的做法是
`assets/css/custom.css`：内容只有三条覆盖（例如 `:root { --radius-lg: 16px; }`），
在 `css.html` 里被拼在主题输出之后，所以靠层叠生效；它里面的 `@import` 也照样过 `css.Build`。

## 指纹、SRI，以及为什么两个文件不合并

生产构建的最后一步是 `fingerprint "sha384"`：它把内容哈希写进文件名，并给出 `integrity` 的值。
于是 `css/bundle.min.573453…css` 这个 URL 与内容一一对应，可以放心长期缓存——
内容变了文件名就变了，不需要靠 `max-age` 去猜。

`integrity` 不是装饰：`crossorigin="anonymous"` 让浏览器按 CORS 规则校验响应体，
哈希对不上就直接拒绝执行。两个文件分开而不是合成一个，是刻意的：
改一行 CSS 不会让 JS 的缓存失效，反过来也一样。
