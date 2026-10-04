+++
title = 'Hugo Scratch'
description = '一个 scratch 站点：把 Hugo 的常用特性全部用上，并且把每一处用法写进可以继续对话的文档里。'
+++

## 这是一个可以一直对话下去的 Hugo 站点

这个仓库由 `hugo new site` 与 `hugo new theme` 生成，**主题是独立的子仓库**，通过 git submodule 挂在 `themes/hugo-scratch-theme`。站点和主题可以各自演进、各自打标签，升级主题不需要动站点内容。

克隆下来就能跑：

```bash
git clone --recurse-submodules https://github.com/hencter/hugo-scratch.git
cd hugo-scratch
hugo server
```

`--recurse-submodules` 不是可选项：少了它，`themes/hugo-scratch-theme` 会是空目录，构建出来的站点没有任何模板。判断方法很简单——如果报 `found no layout file for "html" for kind "page"`，就是主题没拉下来。

## 页面上剩下的部分不是手写的

这一页下面出现的章节卡片、最新文章、标签云和"给 AI 代理"的入口，全部由主题模板从内容树里推导出来：

- 章节卡片来自 `.Sections.ByWeight`，所以新增一个内容目录就会自动多一张卡；
- 最新文章来自 `site.GetPage "/blog"` 的 `RegularPages`，按日期倒序取前三条（条数由 `[params.home] latestCount` 控制）；
- 标签云来自 `site.Taxonomies.tags.ByCount`；
- 机器可读入口来自首页的 output formats，配置里删掉某个格式，这里的链接也会跟着消失，两边不会不同步。

要改这一页，改 `content/_index.md`；要改上面那几块，改 `themes/hugo-scratch-theme/layouts/home.html`。前者是内容，后者是版式，界线是刻意画清的。

## 从哪里开始读

如果你只有十分钟，按这个顺序看三个文件就够理解整个站点了：

1. `config/_default/hugo.toml` —— 站点全部的开关都在这里，逐条带注释。
2. `themes/hugo-scratch-theme/layouts/baseof.html` —— 文档契约：它定义块，页面类型只填 `main`。
3. `content/docs/content/front-matter.md` —— front matter 的字段契约，侧栏、面包屑、上下页和 `pages.json` 都读它。

想直接照着做，从 [快速开始](/docs/start/quick-start/) 进；想知道"到底用了哪些 Hugo 特性"，看 [特性总表](/docs/reference/feature-matrix/)；想把后续工作交给代理，看 [代理工作流](/docs/agents/workflow/)。

## 这个站点的三条底线

**构建不联网，但样式管线要先装一次依赖。** 构建过程没有 `resources.GetRemote`，也不访问任何远端地址。样式表分两段编译：主题的设计系统走 Hugo 内置的 `css.Build`（把全部 `@import` 内联成一份），Tailwind v4 那一段走官方集成 `css.TailwindCSS`——它调用 `npm ci` 装在站点根目录的 CLI，并且只生成渲染结果里真正出现过的工具类；两段再合成一份、压缩、加 `fingerprint` 与 `integrity`。JS 由 Hugo 内置的 esbuild（`js.Build`）打包。所以顺序是：克隆之后 `npm ci`，之后每次构建都可以完全离线。另一个可选依赖是重新生成品牌图的 Python 脚本。

**没有第三方脚本。** 页面只请求同源资源。搜索索引在读者第一次打开搜索框时才抓取；`youtube` 短代码刻意只渲染一个链接而不是 iframe，这样在读者决定离开之前，不会有任何请求发往第三方。统计默认关闭，而且只在生产环境生效。

**警告即失败。** 仓库用的严格构建是：

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

`--panicOnWarning` 意味着第一条 WARNING 就会中断构建。一个被弃用的配置键、一个没人调用的模板、一条解析不到页面的站内链接、一张找不到的图片，都会让构建失败——这是刻意的：这些问题在预览里看着无害，上线后就是死链和失效的结构化数据。
