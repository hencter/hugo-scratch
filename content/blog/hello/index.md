+++
title = '为什么从脚手架开始'
description = '从 hugo new site 与 hugo new theme 出发，看脚手架真正生成了哪些文件，以及主题为什么被拆成独立仓库。'
date = 2026-01-12
authors = ['Hencter Lew']
tags = ['Hugo', '主题']
categories = ['工程实践']
series = ['从零搭一个 Hugo 站点']
+++

## 脚手架给出的东西比想象中多

`hugo new site` 只做一件事：把空目录摆好，并且写一份最小的 `hugo.toml`。
真正给出内容的第二步是 `hugo new theme`，它生成的主题骨架已经是**当前模板系统**，
不是很多人记忆里的那份旧布局：

{{< filetree >}}
example/
├── archetypes/
│   └── default.md          # hugo new content 的模板
├── assets/                 # 需要处理的资源（Hugo Pipes 的入口）
├── content/
├── data/
├── hugo.toml
├── i18n/
├── layouts/
├── static/                 # 原样拷贝的文件
└── themes/
    └── mytheme/
        ├── hugo.toml
        ├── archetypes/default.md
        ├── assets/
        │   ├── css/
        │   │   ├── main.css
        │   │   └── components/{header,footer}.css
        │   └── js/main.js
        ├── content/            # 示例内容：上线前应删掉
        ├── layouts/
        │   ├── baseof.html
        │   ├── home.html  page.html  section.html
        │   ├── taxonomy.html  term.html
        │   └── _partials/
        │       ├── head.html  header.html  footer.html  menu.html  terms.html
        │       └── head/{css,js}.html
        └── static/favicon.ico
{{< /filetree >}}

三件容易被忽略的事：模板放在 `layouts/` 根下，`baseof.html` 靠 `{{/* block */}}`
给子模板留位置；`_partials/` 与 `_shortcodes/` 的**下划线前缀是保留前缀**，
写错了不会被报错，只会找不到模板；`assets/` 与 `static/` 的区别是前者要过管线、后者原样拷贝。

第二件：主题的 `hugo.toml` 里有 `[module.hugoVersion]`（`min = '0.146.0'`），
并且带了 `[[menus.main]]` 三个演示菜单项。菜单是**会合并**的配置之一，
不删就会出现在你的导航里。

第三件：主题骨架里带了 `content/_index.md` 和 `content/posts/`。
那是演示内容，不是模板的一部分；这个站点的主题仓库里没有 `content/` 目录。

## 骨架里的两条构建管线

脚手架最值钱的部分其实是 `layouts/_partials/head/css.html` 与 `head/js.html`：
它们已经把 Hugo Pipes 的两条管线写好了。CSS 一侧是

```go-html-template
{{- with resources.Get "css/main.css" }}
  {{- $opts := dict
    "minify" (cond hugo.IsDevelopment false true)
    "sourceMap" (cond hugo.IsDevelopment "linked" "none")
  }}
  {{- with . | css.Build $opts }}
    {{- with . | fingerprint }}
      <link rel="stylesheet" href="{{ .RelPermalink }}" integrity="{{ .Data.Integrity }}" crossorigin="anonymous">
```

`css.Build` 把 `main.css` 里那几条 `@import` 内联成一个文件，
`cond hugo.IsDevelopment` 让开发构建保留 source map、生产构建才压缩。
JS 一侧的差别只有函数名：`js.Build` 交给 esbuild，把 `import` 图打包成一个文件。

脚手架给出的两条管线里，只有样式表那一段后来多了外部依赖：设计系统仍由 `css.Build` 内联，脚本仍由 `js.Build` 打包，两者都在 `hugo` 二进制里；Tailwind v4 那一段走 `css.TailwindCSS`，会调用站点根目录下由 `npm ci` 装好的 CLI。

## 这个站点在骨架上改了什么

| 位置 | 脚手架给的 | 这里改成了 |
| --- | --- | --- |
| `head/css.html` | 只构建主题的 `main.css` | 再构建项目自己的 `css/custom.css`，用 `resources.Concat` 拼在后面 |
| `head/js.html` | `js.Build $opts` | 多传一个 `params`，于是 esbuild 里多出一个 `@params` 虚拟模块 |
| `fingerprint` | 默认 `sha256` | `fingerprint "sha384"`，与 `integrity` 属性对上 |
| 缺资源时 | `with` 静默跳过 | 加 `errorf`，让构建直接失败 |

多出来的那一行 `params` 是双语站点能工作的前提之一，下一篇再展开。
真正需要记住的是最后一行：**`resources.Get` 找不到文件时返回 nil，`with` 会安静地跳过整段**，
于是页面渲染出来是没样式的，而构建日志一句话都不说。

## 主题为什么是独立子仓库

把主题放在 `themes/hugo-scratch-theme` 而不是 `layouts/` 下面，只为了三件事：

1. **改动可以分开追溯。** 只改样式或模板的提交落在主题仓库里，主题仓库的 `main` 就是主题的历史；
   项目仓库里只留下父提交指针的一行变化。
2. **主题可以被第二个站点复用。** 仓库地址是
   <https://github.com/hencter/hugo-scratch-theme>；一个站点 `theme = ['hugo-scratch-theme']`
   就挂上来，重复的部分不需要复制粘贴。
3. **本地开发仍然是即时反馈。** `hugo server` 直接读这个目录，改一行 CSS 立刻生效，
   不需要发一个版本才能看到效果。

代价是克隆和部署时多一步：部署时先取主题再构建，
`git clone --recurse-submodules https://github.com/hencter/hugo-scratch.git`
一次把两边都取下来。这一步在[快速开始](/docs/start/)里也是这么写的。
主题只贡献 `layouts/`、`assets/`、`i18n/`、`static/` 和它的 `params`，
内容与站点配置始终归项目仓库所有——这也是为什么主题的 `hugo.toml` 里写 `baseURL` 没有用。
