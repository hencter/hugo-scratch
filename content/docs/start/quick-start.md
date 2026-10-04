+++
# Alias paths are relative to the site root, and Hugo adds the language prefix
# itself — so the English twin declares the same path and lands at /en/docs/quick-start/.
aliases = ['/docs/quick-start/']
title = '快速开始'
linkTitle = '快速开始'
description = '确认 Hugo 版本、带子模块克隆仓库、启动开发服务器，然后验证你看到的是对的页面。'
date = 2026-01-10
weight = 10
difficulty = 'beginner'
estimatedTime = 12
prerequisites = ['/docs/start/']
outcomes = ['在本地把站点跑起来', '认出子模块没拉下来时的那条报错', '知道下一站该读哪一页']
tags = ['Hugo']
+++

这一页只做一件事：让这个仓库在你的机器上渲染出来。它不解释任何设计决策，那些留给后面的章节。整个过程只有三条命令，但其中一条容易被复制错，所以下面会说明它为什么不能省。

## 开始之前

你需要一个 Hugo 可执行文件，版本 **0.146 或更高**。这是主题自己声明的下限——`themes/hugo-scratch-theme/hugo.toml` 里的 `[module.hugoVersion]` 写着 `min = '0.146.0'`。低于它时 Hugo 会打出一条 `Module "hugo-scratch-theme" is not compatible with this Hugo version` 的**警告**，然后照常构建完成；所以裸 `hugo` 不会拦住你，而本仓库的严格构建（`--panicOnWarning`）会。构建本文档所用的版本是 {{< version >}}。

除此之外还需要 `git`、一个文本编辑器，以及一次用来装依赖的网络访问。样式表分两段编译：主题的设计系统走 Hugo 内置的 `css.Build`（不需要 Node），Tailwind v4 那一段走官方集成 `css.TailwindCSS`，而它调用的是 npm 装在**站点根目录**的 Tailwind CLI——所以克隆之后要先 `npm ci`。脚本那一段始终由 Hugo 内置的 esbuild 打包，不需要额外工具。

```bash
hugo version
```

## 克隆，并且带上子模块

主题不是一个内嵌的目录，而是一个 git 子模块，挂在 `themes/hugo-scratch-theme`。

{{% steps %}}
{{% step "带子模块克隆" %}}
```bash
git clone --recurse-submodules https://github.com/hencter/hugo-scratch.git
cd hugo-scratch
```
`--recurse-submodules` 让 git 在克隆主仓库之后立刻进入子模块并拉取它。少了这个参数，`themes/hugo-scratch-theme` 会是一个空目录。
{{% /step %}}
{{% step "确认子模块真的在" %}}
```bash
ls themes/hugo-scratch-theme
```
你应该看到 `hugo.toml`、`layouts/`、`assets/` 这些条目。如果这个目录是空的，站点还没有准备好。
{{% /step %}}
{{% step "已经克隆过了？补一次" %}}
在仓库根目录执行：
```bash
git submodule update --init --recursive
```
{{% /step %}}
{{% /steps %}}

{{< warning title="子模块缺失时看到的现象" >}}
如果主题目录是空的，Hugo 不会说"子模块没拉下来"，它只会为每个页面报类似 `found no layout file for "html" for kind "page"` 的错误，或者构建出一个完全没有样式、没有导航的页面。看到"找不到布局文件"时，先检查 `themes/hugo-scratch-theme` 是否为空，再去看模板。
{{< /warning >}}

## 安装样式管线的依赖

CSS 的其中一段由 Tailwind v4 编译，而 `css.TailwindCSS` 调用的是 npm 装在站点根目录的 CLI，不是 Hugo 内置的东西：

```bash
npm ci
```

版本已经钉在 `package-lock.json` 里，所以用 `npm ci` 而不是 `npm install`：它按锁文件精确还原，结果是可重复的。跳过这一步，构建会停在 Tailwind 那一段，报的是找不到 `tailwindcss` 可执行文件——注意这条报错不会指向任何页面，容易被误判成模板问题。

## 启动开发服务器

```bash
hugo server
```

服务器默认监听 1313 端口，打开终端里打印的那个地址即可。它会在内存里构建，把 `public/` 留给你自己的正式构建使用，并且在文件改动时自动重建、通过浏览器重载页面——所以整节工作流是"改文件、保存、看浏览器"，不需要重启服务器。

{{< tip >}}
如果 1313 被别的进程占用，`hugo server` 会换一个端口并把新地址打印出来，读终端输出比死记端口号可靠。想固定端口，可以用 `hugo server --port 1414`。
{{< /tip >}}

## 验证

站点点开后，按这个顺序确认三件事，它们各自失败时的含义完全不同：

- 首页有标题、导航栏、语言切换器，样式正常。如果只有纯文本，是主题没加载；如果布局对但配色不对，是样式表的问题，不是这一页要处理的范围。
- 顶栏的"文档"能进入[文档](/docs/)首页，左侧栏出现章节列表。侧栏只对 `[params.nav] sidebarSections` 里列出的章节显示，`docs` 就在其中。
- 打开 `content/docs/start/quick-start.md`，改一句正文并保存，浏览器应该自动刷新。热重载不生效的话，后面每次改动都要手动重启服务器。

想确认 Hugo 到底把哪些文件当成内容，用 `hugo list all` 打印清单；想确认某次改动是否影响构建，用 `hugo --logLevel warn` 做一次一次性构建。

## 接下来

本地能跑之后，先读[目录结构](/docs/start/directory-structure/)，它标出了 `public/`、`resources/` 这些不该手改的目录。然后再读[配置](/docs/configuration/)，理解这个站点为什么不把 `hugo.toml` 放在根目录，以及主题、`_default`、环境配置三层是怎么合并的——后面每一页提到的配置项，出处都在那里。

## 交付前的门禁

{{< include "build-gate" >}}

## 参考

{{< docref "getting-started/"  >}}
