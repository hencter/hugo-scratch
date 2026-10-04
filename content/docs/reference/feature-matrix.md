+++
title = '特性总表'
linkTitle = '特性总表'
description = '这个站点用到的每一条 Hugo 特性：它在哪实现、怎么验证、以及它对应的文档页。'
date = 2026-03-01
weight = 10
difficulty = 'intermediate'
estimatedTime = 20
prerequisites = ['/docs/start/directory-structure/']
outcomes = ['能按表定位任意一条特性的实现位置', '知道每条特性的验证命令', '知道哪些说法是自己验证过的、哪些是推断的']
tags = ['参考', '特性']
+++

这张表是「这个站点到底用了什么」的**唯一入口**。每一行都指到一个真实文件或一条真实命令；「验证」一列写的是我实际跑过、并且看到对应产物的方式，不是推测。

约定：**实现位置**里的路径都是仓库根目录起的相对路径；**验证**里的命令都在仓库根目录执行。

## 站点骨架与配置

| 特性 | 实现位置 | 验证 |
| --- | --- | --- |
| `hugo new site` / `hugo new theme` 脚手架 | 仓库根 + `themes/hugo-scratch-theme/` | `hugo new theme <name>` 生成的就是这个结构 |
| 主题独立成仓库、用 submodule 挂载 | `.gitmodules`、`themes/hugo-scratch-theme` | `git submodule status` |
| 配置目录分层（主题 → `_default` → 环境） | `config/_default/*.toml`、`config/production/hugo.toml` | `hugo config` |
| 显式 module mounts（含 `hugo_stats.json`） | `config/_default/hugo.toml` 的 `[module]` | 删掉 `layouts` 那条挂载，构建会找不到模板 |
| 站点级 cascade（按路径改 sitemap 权重） | 同文件的 `[[cascade]]` | `grep changefreq public/zh-cn/sitemap.xml` |
| Git 联动日期（`enableGitInfo` + `:git`） | `config/_default/hugo.toml` 的 `[frontmatter]` | 页面上「更新于」等于该文件的最后提交日 |
| `buildStats` 产出 `hugo_stats.json` | `[build.buildStats]` | 构建后读 `hugo_stats.json` 里的 `classes` 数组 |

## 内容管理

| 特性 | 实现位置 | 验证 |
| --- | --- | --- |
| 分支包 / 叶子包 / 无头包 | `content/docs/**`、`content/blog/*/index.md` | `hugo list all` |
| 页面资源与图片处理 | `content/blog/hugo-pipes/cover.png` | 页面上图片带 `width`/`height` |
| 内容适配器（从数据生成页面） | `content/changelog/_content.gotmpl` + `data/changelog.toml` | `public/changelog/v1-0-0/index.html` 存在 |
| 三种分类法 | `[taxonomies]` = tag / category / series | `public/tags/`、`public/categories/`、`public/series/` |
| 前置元数据字段契约 | `themes/hugo-scratch-theme/archetypes/docs.md` | `hugo new content docs/x.md` 会套用 |
| 别名与重定向 | `aliases` 前置字段 | 生成的别名页是 `<meta http-equiv="refresh">` |
| 分页（每页 2 条，刻意调小） | `[pagination] pagerSize` | `public/blog/page/2/index.html` |
| 按年分组的列表 | `content/blog/_index.md` 的 `groupByYear` | 博客列表出现 `<h2 id="year-2026">` |
| 草稿 / 未来 / 过期与 `notice` 横幅 | `layouts/_partials/banner.html` | 给一页加 `notice = "…"` 再构建 |
| 相对页面导航（上下页） | `layouts/_partials/page-nav.html` | 页脚的上一篇/下一篇 |

## 模板与渲染

| 特性 | 实现位置 | 验证 |
| --- | --- | --- |
| `baseof` 文档契约 + `main` 块 | `themes/hugo-scratch-theme/layouts/baseof.html` | 每个页面类型只写 `{{ define "main" }}` |
| 页面类型模板（home/page/section/taxonomy/term/404） | 同目录同名文件 | 六种页面都存在对应产物 |
| 返回值的 partial | `layouts/_partials/resolve-image.html` | 返回 `{url,width,height,resource}` |
| 返回布局判定的 partial | `layouts/_partials/layout/flags.html` | 返回 `{sidebar,toc}`，`baseof` 用它决定栅格 |
| 内联 partial（`define` 写在 partial 里） | `layouts/_partials/sidebar.html` 等 | 侧栏树、菜单、目录、列表行都用这个模式 |
| 短代码：标准记法（`.Inner` 是原文） | `layouts/_shortcodes/note.html` 等 | 见 [短代码](/docs/content/shortcodes/) |
| 短代码：Markdown 记法（`.Inner` 已渲染） | `tabs.html` / `steps.html` / `columns.html` | 同上 |
| 短代码间共享 `.Store` | `tabs.html` 读、`tab.html` 写父级 store | 标签页按钮是服务端渲染出来的 |
| 覆盖内置短代码 | `figure.html`、`youtube.html` | 站点的同名文件优先 |
| 七个渲染钩子 | `layouts/_markup/render-*.html` | 见 [渲染钩子](/docs/content/render-hooks/) |
| 表格渲染钩子包一层滚动容器 | `render-table.html` | 页面上表格外层是 `.table-wrap` |
| 渲染钩子做构建期校验 | `render-link.html`、`render-image.html` | 写一条断链，构建会警告 |
| `templates.Defer` 延迟渲染 | `layouts/_partials/head.html` 里的 CSS 引用 | Tailwind 需要渲染完成后的 `hugo_stats.json` |

## 资源管线：CSS 与 JS

| 特性 | 实现位置 | 验证 |
| --- | --- | --- |
| 官方 Tailwind 集成（`css.TailwindCSS`） | `layouts/_partials/head/css.html` + `assets/css/tailwind.css` | 产物里能找到 `.mt-6`、`.flex` 等只用过一次的工具类 |
| Tailwind 的扫描源是渲染结果 | `@source "hugo_stats.json"` | 改一个模板里的类名，重新构建后产物会变 |
| 分层（theme < components < utilities） | `head/css.html` 前置 `@layer` 声明 | 产物第一个 `@layer` 就是顺序声明 |
| 显式跳过 Tailwind preflight | `assets/css/tailwind.css` 的注释与导入 | 列表仍有圆点；主题自带 reset 生效 |
| 设计系统用 `css.Build` 合并 `@import` | `assets/css/design-system.css` | 产物只有一个 css 文件 |
| 站点自有样式追加（不覆盖主题） | 站点 `assets/css/custom.css` | 改一处圆角，产物里能找到 |
| `resources.Concat` 合并两段产物 | `head/css.html` | 最终仍是一个 `<link>` |
| `minify` + `fingerprint "sha384"` + SRI | `head/css.html`、`head/js.html` | 产物标签上有 `integrity` 与 `crossorigin` |
| `js.Build`（内置 esbuild）打包 ES 模块 | `assets/js/main.js` + `modules/*.js` | 一个 `<script defer>` 覆盖全部交互 |
| `@params` 虚拟模块注入配置与译文 | `head/js.html` 的 `params` 选项 + `main.js` 的 `import * as params from '@params'` | 两种语言各得一份哈希不同的 JS |
| 明暗主题不闪烁（内联引导脚本） | `head/theme-init.html` + `assets/css/chroma-*.css` | 产物 `<html>` 上同时有 `data-theme` 与类名 |

## 输出格式与机器可读出口

| 特性 | 实现位置 | 验证 |
| --- | --- | --- |
| 自定义媒体类型与输出格式（主题声明） | `themes/hugo-scratch-theme/hugo.toml` | `hugo config` 里有 `outputformats` |
| 哪些页面产出哪些格式（站点决定） | `config/_default/hugo.toml` 的 `[outputs]` | 见下表 |
| 每页的 Markdown 孪生页 | `layouts/page.md`、`layouts/list.md` | `/docs/start/quick-start/index.md` |
| `llms.txt` | `layouts/home.llms.txt` | `/llms.txt` |
| 客户端搜索索引 | `layouts/home.search.json` | `/search.json` |
| 面向代理的页面清单 | `layouts/home.pages.json` | `/pages.json` |
| 合法 YAML 前置元数据 | `transform.Remarshal "yaml"` | 孪生页开头是可直接解析的 YAML |
| 自定义 sitemap（含语言互链与权重） | `layouts/sitemap.xml` | `/zh-cn/sitemap.xml` 里有 `xhtml:link` |
| 生成式 robots.txt（按环境放行） | `layouts/robots.txt` | 生产构建 `Allow: /` |
| 自定义 RSS（受 `[services.rss] limit` 限制） | `layouts/rss.xml` | `/index.xml` |

| 页面类型 | 输出格式 |
| --- | --- |
| home | `html` `rss` `md` `llms` `search` `pages` |
| section / taxonomy / term | `html` `rss` `md` |
| page | `html` `md` |

## SEO 与语义化

| 特性 | 实现位置 | 验证 |
| --- | --- | --- |
| 绝对 canonical（翻页指向自身） | `layouts/_partials/head/meta.html` | 看 `public/blog/page/2/index.html` |
| 非生产环境一律 noindex | 同文件，判断 `hugo.IsProduction` | `hugo server` 下看任意页面 |
| Open Graph + Twitter Card | `head/opengraph.html` | `og:image` 是绝对地址且带宽高 |
| JSON-LD `@graph` | `head/schema.html` | `<script type="application/ld+json">` 里是**对象**不是字符串 |
| hreflang 与 `x-default` | `head/alternates.html` | 每页三条 `rel="alternate"` |
| 站长验证 meta（默认零输出） | `head/verification.html` | 留空时页面上没有该标签 |
| 语义化地标与跳转链接 | `layouts/baseof.html`、`header/footer` | `skip-link`、`aria-current`、`<time datetime>` |
| 打印样式表 | `assets/css/print.css` | 打印预览里导航消失、外链补出 URL |
| 尊重「减少动态效果」 | `assets/css/base.css` | `prefers-reduced-motion` 分支 |

## 多语言与 i18n

| 特性 | 实现位置 | 验证 |
| --- | --- | --- |
| 单内容树 + `.en.md` 配对 | `content/**/*.en.md` | 页面上有三个 hreflang |
| 每语言菜单 | `config/_default/menus.zh-cn.toml`、`menus.en.toml` | 主菜单中英文不同 |
| 词条表（两语言键必须一致） | `themes/hugo-scratch-theme/i18n/*.toml` | `--printI18nWarnings` 静默 |
| 复数形式 | `readingTime`、`pageCount` 等键 | `T "key" 数字` |
| 每语言日期格式 | `config/_default/languages.toml` 的 `[<lang>.params]` | 中文页面显示「2026 年 3 月 24 日」 |
| 语言切换不产生 404 | `layouts/_partials/lang-switcher.html` | 无译文的页面回退到该语言首页 |
| 每语言 JS 包（译文进打包产物） | `head/js.html` 的 `params.i18n` | `public/js/` 下有两个哈希 |

## 导航与交互（无第三方脚本）

| 特性 | 实现位置 | 验证 |
| --- | --- | --- |
| 主菜单与激活态 | `layouts/_partials/menu.html` | 当前项带 `aria-current="page"` |
| 章节侧栏（只展开当前路径） | `layouts/_partials/sidebar.html` | 只有当前链路的子节点展开 |
| 本页目录与滚动高亮 | `layouts/_partials/toc.html` + `modules/toc.js` | 滚动时目录项出现 `aria-current` |
| 面包屑 | `layouts/_partials/breadcrumbs.html` | 与 JSON-LD 的 BreadcrumbList 一致 |
| 客户端搜索（首次打开才拉索引） | `modules/search.js` | 网络面板只在打开搜索框后出现 `search.json` |
| 明暗主题切换 | `modules/theme.js` + `theme-toggle.html` | 刷新后仍保持选择 |
| 标签页键盘导航 | `modules/tabs.js` | 左右方向键切换面板 |
| 代码复制 | `modules/copy.js` + `render-codeblock.html` | 点击后按钮文案变「已复制」 |
| 返回顶部 | `modules/back-to-top.js` | 滚过一屏后出现 |
| 移动端导航（纯 CSS 切换） | `modules/nav.js` | 无 JS 时菜单仍然存在且可点 |

## 构建、验证与部署

| 特性 | 实现位置 | 验证 |
| --- | --- | --- |
| 严格构建（警告即失败） | `AGENTS.md`、README | 下面的命令 |
| 并行写作互不干扰的隔离构建 | `--cacheDir` + `-d` | 见 [代理工作流](/docs/agents/workflow/) |
| CI：装依赖 → 严格构建 → 部署 | `.github/workflows/*.yml` | 见 [部署到 GitHub Pages](/docs/deploy/github-pages/) |
| 构建离线可跑 | 无 `resources.GetRemote`、无网络调用；Tailwind CLI 由 `npm ci` 预装 | 装完依赖后断网执行严格构建 |

## 怎么验证整件事

一条命令，退出码为 0 才算过：

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

它同时挡住五类问题：被弃用的配置键或模板方法（`--panicOnWarning`）、两个页面写到同一个目标路径（`--printPathWarnings`）、没人调用的模板（`--printUnusedTemplates`）、缺翻译（`--printI18nWarnings`）、以及渲染钩子报出的断链与缺图。

文档还建议在发布前跑一遍站点审计，它会把两类**默认被静默**的问题变成可见的：

```bash
HUGO_MINIFY_TDEWOLFF_HTML_KEEPCOMMENTS=true HUGO_ENABLEMISSINGTRANSLATIONPLACEHOLDERS=true hugo --ignoreCache
grep -rn "MISSING_TRANSLATION" public/       # 缺翻译占位
grep -rn "raw HTML omitted" public/          # unsafe 没开时的 HTML 被吃掉
```

第三条要在产物里搜 Hugo 的短代码占位符，完整字符串是 H&#xfeff;AHAHUGOSHORTCODE。这里只能这样写：完整的占位符前缀一旦出现在**正文**里，构建会直接中止并报 `illegal state in content; shortcode token missing end delim`，而且报错指向正在渲染的那一页，不一定是你写它的那一页。写成 HTML 实体之后，渲染出来的产物里也不含那个字面串。

{{< warning >}}
这三条 grep 都可能命中文档本身——这一页就写着 `MISSING_TRANSLATION` 与 `raw HTML omitted`，[导航](/docs/configuration/navigation/)和 [Markdown](/docs/content/markdown/) 两页也各写过其中一条。仓库的 CI 因此按路径排除了记录这些字符串的页面；审计命令要么排除文档，要么接受第一次运行就会因为自己的说明文本而失败。
{{< /warning >}}

最后，构建产物本身才是「页面到底渲染了没有」的证据：`public/` 里存在对应目录，才算这一页真的存在——控制台什么都没说，不代表它渲染了。
