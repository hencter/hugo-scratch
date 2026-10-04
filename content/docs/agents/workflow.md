+++
title = '用对话维护这个站点'
linkTitle = '协作流程'
description = '代理接手时的入口、事实来源与验证步骤，以及内容作者必须遵守的红线。'
date = 2026-03-02
weight = 10
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/start/directory-structure/', '/docs/']
outcomes = ['知道代理先读哪个文件、去哪里找一页的事实', '用严格构建加阅读渲染产物证明一次改动', '分清源码与 public/、resources/ 这类输出', '用受支持的开关加页面级目录或横幅']
tags = ['智能体']
+++

## 两个入口：AGENTS.md 与页面 front matter

仓库根的 `AGENTS.md` 是唯一需要先读的文件。它是一份给代理看的简报，应该保持短而可执行，只写真正会改变行为的内容：目录地图、构建命令、不能碰的路径、内容规则。站点结构或规则变了就改它，不要在下游文档里另起一份说明——两份说法迟早会互相矛盾，而代理只会读其中一份。

第二类入口是每一页自己的 front matter。一页的成本和收益写在字段里，而不是正文里：

```toml
weight = 10
difficulty = 'intermediate'
estimatedTime = 25
prerequisites = ['/docs/start/quick-start/', '/docs/configuration/site-config/']
outcomes = ['说清 baseURL 为什么必须与站点的真实访问地址一致', '把仓库的 Pages 源切成 GitHub Actions']
```

`layouts/_partials/facts.html` 把这些字段渲染成页面顶部的信息面板，`/pages.json`（见[输出格式](/docs/templates/output-formats/)）携带同一批值，所以人和代理看到的是同一个答案。`prerequisites` 用站点根相对路径书写，面板会拿它去 `site.GetPage`：解析成功就显示目标页的短标题和真实链接，解析失败就把字符串原样打印出来。

{{< note >}}
TOML 里所有标量都要写在第一个 `[table]` 头之前。一个裸键写在表头下面会静默地变成那张表的成员，字段于是从渲染结果里消失——这类错误没有任何报错。
{{< /note >}}

## 代理如何证明一次改动

一条命令，加一次阅读产物。

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

退出码 0 意味着构建通过而且没有任何警告——包括没有指向不存在页面的站内链接（`layouts/_markup/render-link.html` 会对这类链接发出 WARNING），也包括没有哪个模板是任何东西都到不了的。但退出码只证明"没坏"，不证明"改对了"。第二步是读渲染结果：`public/docs/start/quick-start/index.html` 里应该出现你新写的那句话；同一页的 Markdown 输出 `public/docs/start/quick-start/index.md` 必须与之一致。判断某页有没有渲染，看 `public/` 下有没有对应目录，不要因为控制台没报错就认为它进了构建。

{{< tip >}}
`hugo list all` 列出 Hugo 认为自己拥有的全部页面。页数对不上时，通常是一页的 front matter 让它没进构建，而不是模板坏了。
{{< /tip >}}

## 输出不是源码

`public/`、`resources/`、`hugo_stats.json` 以及构建锁 `.hugo_build.lock` 都是产物或状态文件，`.gitignore` 里也是这么记的。它们不是"暂时改一下也行"的地方：

- `public/` 是构建目标目录。下一次 `hugo` 会重写它，手改的内容会消失；更糟的是你会读到一份不代表源码的页面，并据此得出错误结论。
- `resources/_gen/` 是 Hugo Pipes 处理过的资源缓存。删掉它没问题，改它等于和下一次构建对赌。
- `hugo_stats.json` 只在构建被要求输出统计时生成，是给外部工具读的产物，不是配置。

## 内容红线

- **想展示短代码语法就必须转义。** 写成 `{{</* note */>}}` … `{{</* /note */>}}` 和 `{{%/* tabs */%}}` … `{{%/* /tabs */%}}`。正文里未转义的短代码起始分隔符（两个左花括号，后面跟 `<` 或 `%`）会被真的执行，**代码块里的也不例外**，围栏保护不了你。
- **永远不要写 Hugo 的内部占位符。** 渲染短代码时，Hugo 先把调用替换成一个全大写的占位字符串，Markdown 跑完再换回来。这个字符串一旦被写进内容（最常见的来源是从构建日志或产物片段里复制），渲染会以 `illegal state in content` 失败，而 Hugo 会把错误算到当时正在渲染的那一页头上，指向一个其实没问题的文件。
- **正文从 `##` 开始。** 页面的 `<h1>` 由模板输出，正文里再写一个 `#` 就是重复标题。
- **双语成对。** 改动落到 `name.md` 就必须落到同目录的 `name.en.md`，标题层级、代码块和短代码调用保持一致。
- **站内链接用站点根相对路径**（如 `/docs/start/quick-start/`），并且必须指向存在的页面，否则严格构建直接失败。

## 页面级目录与横幅

这两个需求各有受支持的开关，不要自己写 HTML：

- **页面内目录**：在正文里调用 `{{</* toc */>}}`，`layouts/_shortcodes/toc.html` 会就地打印 Hugo 为这一页生成的 `.TableOfContents`；包含哪些级别由配置里的 `[markup.tableOfContents]` 决定。它和左侧栏那套是两条不同的实现：`layouts/_partials/toc.html` 从 `.Fragments` 重建目录，带滚动高亮，并受 `[params.ui] tocMinHeadings` 约束。不想要侧栏目录时，在 front matter 写 `toc = false`。
- **页面横幅**：在 front matter 写 `notice = '这一页正在重写'`，`layouts/_partials/banner.html` 会在页面顶部渲染一条提示。`notice` 是字面字符串，不是 i18n 键——写什么显示什么。草稿页在被 `-D` 构建时还会额外显示一条草稿提示。

这一页自己就调用了一次目录短代码。下面这段是它在正文位置打印出来的结果，标题和左侧栏那份来自 `.Fragments` 的目录是同一批：

{{< toc >}}

## 一次真实请求的完整路径

请求原文：「快速开始那一页要写明克隆必须带 `--recurse-submodules`，并在页面顶部加一条提示。」

1. **定位。** 代理先读 `AGENTS.md`，再读 `content/docs/start/quick-start.md` 的 front matter，确认要改的是哪一页、这一页的门槛和产出是什么。
2. **落文件。** 改动落在 `content/docs/start/quick-start.md` 与 `content/docs/start/quick-start.en.md` 两个文件，标题层级和代码块两份保持一致。横幅写在 front matter 的 `notice` 字段里，不是正文里的自定义 HTML。如果这条规则本身在 `AGENTS.md` 里没写清，顺便把它补上——那是代理下一次会读的地方。
3. **证明。** 跑严格构建拿到退出码 0；读 `public/docs/start/quick-start/index.html`，确认新句子和带 `class="banner"` 的提示条都在；再读 `public/docs/start/quick-start/index.md`，确认 Markdown 输出与页面一致；最后用 `hugo list all` 确认页数没变——一次纯内容改动不应该改变站点有多少页。

## 同一个门禁，只有一份定义

{{< include "build-gate" >}}
