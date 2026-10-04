+++
title = 'CSS 构建管线'
linkTitle = 'CSS'
description = '两段编译合成一份样式表：css.Build 内联主题的设计系统、Tailwind v4 生成真正用到的工具类、站点的 custom.css 在哪一步拼进来，以及明暗主题怎样接到 Chroma 的代码配色上。'
date = 2026-02-25
weight = 10
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/', '/docs/assets/']
outcomes = ['读懂 head/css.html 的两段编译和拼接顺序', '知道站点自己的样式该写进哪个文件', '能重新生成明暗两套 Chroma 样式表']
tags = ['CSS']
+++

`layouts/_partials/head/css.html` 把两段互不相干的编译拼成一份样式表：主题的设计系统走 `css.Build`，Tailwind 走 `css.TailwindCSS`，最后一起压缩、加指纹、带上 SRI。整条管线不到九十行模板，却决定了几件事——浏览器发几个请求、站点改动会不会被主题更新冲掉、以及切换明暗主题时连代码块会不会跟着变色。

## 两段编译，一份样式表

主题的设计系统入口是 `assets/css/design-system.css`，它自己几乎不写规则，只有十条 `@import`：

{{< filetree >}}
themes/hugo-scratch-theme/assets/css/
  design-system.css   ← 设计系统入口，只写 @import
  tokens.css          ← 颜色、间距、字体等自定义属性
  chroma-light.css    ← 代码高亮的浅色规则（自动生成）
  chroma-dark.css     ← 代码高亮的深色规则（自动生成）
  base.css            ← 重置、元素默认值、无障碍基元
  layout.css          ← 栅格与三栏布局
  components.css      ← 按钮、卡片、目录等组件
  content.css         ← .prose 里的正文排版
  blocks.css          ← 首页各区块
  releases.css        ← 更新日志页的样式
  print.css           ← @media print
  tailwind.css        ← Tailwind v4 的入口（第二段）
{{< /filetree >}}

第一段是设计系统：`resources.Get "css/design-system.css"` 与站点自己的 `css/custom.css` 先经 `resources.Concat` 合成 `css/design-system-bundle.css`，再交给 `css.Build` 内联其中的全部 `@import`，结果由模板包进 `@layer components { … }`。第二段是 Tailwind：`resources.Get "css/tailwind.css"` 交给 `css.TailwindCSS`，由 Tailwind v4 的 CLI 展开 `tailwindcss/theme.css` 与 `tailwindcss/utilities.css`，并只生成真正被用到的工具类。

两段最后拼在一起，顺序是固定的：

```text
@layer theme, components, utilities;   ← 单独的资源，必须排在最前
        ↓
[tailwind.css 的产物：theme + utilities 层]  →  [@layer components { 设计系统 }]
        ↓
resources.Concat "css/bundle.css"
        ↓
（生产）minify → fingerprint "sha384"
        ↓
<link rel="stylesheet" href="…/css/bundle.min.<hash>.css" integrity="sha384-…" crossorigin="anonymous">
```

层序声明由模板作为独立资源插在文件最前面，而不是写在 `tailwind.css` 里：`@layer` 语句只有在它出现在所有分层规则之前才决定优先级，写在入口文件里会被拼到中部，压缩之后位置更难保证——那样 `components` 层会落到 `utilities` 层下面，`class="mt-0"` 就会输给 `.prose p { margin }`，而在任何一条规则里都看不出原因。

## 为什么要拆成两段

一个入口文件装不下两种导入。Tailwind 的 CLI 以**运行 Hugo 的目录**（项目根）为基准解析 `@import`，所以它认识 `tailwindcss/theme.css` 这种裸模块名；而 `tokens.css` 这种相对路径只有 Hugo 自己的内联器认识，它以资源树为基准，因此站点提供的 `assets/css/custom.css` 也能被找到。拆成两个入口，等于把每种导入交给认识它的解析器——反过来把两种导入塞在一起，构建就会停在 `Can't resolve 'tokens.css' in '<项目根>'`，报错本身也说明了基准是运行目录而不是样式表所在目录。

## custom.css：站点与主题的分界线

主题和站点的 `assets/` 在查找时是同一个命名空间，但合并是**文件级**的：同一个路径只有一份，站点那份静默覆盖主题那份。站点的样式改动因此全部写进 `assets/css/custom.css`，它在第一段里跟在 `design-system.css` 后面被拼接，于是和设计系统同属 `components` 层，靠层叠顺序生效，而不是靠 `!important`。

{{< warning title="不要放一个自己的 assets/css/design-system.css" >}}
那会整体替换主题的样式表：构建成功、页面渲染、样式全丢，下一次主题更新还会带走所有修复。`custom.css` 演示了正确的做法——覆盖自定义属性而不是重写规则：

```css
:root {
  --radius-lg: 16px;
}
```

一个变量改掉，卡片、代码块和提示框一起跟着变，因为它们的圆角读的是同一个 token。
{{< /warning >}}

{{% steps %}}
{{% step "写进 custom.css" %}}
小改动直接追加到 `assets/css/custom.css`。它已经被管线读取，不需要动任何模板。
{{% /step %}}
{{% step "单独开一个文件，再 import 进来" %}}
新建 `assets/css/home.css`，在 `custom.css` 顶部写 `@import "home.css";`。`css.Build` 对 `custom.css` 做的是同样的内联处理，所以它并进第一段，最终产物仍然只有一个文件。
{{% /step %}}
{{% step "确认产物" %}}
`hugo --ignoreCache` 之后看 `public/css/` 下文件名的哈希是否变化。看不到变化时先检查文件是不是放在了 `static/css/`——那是一个会被原样拷贝、完全绕开管线的目录。
{{% /step %}}
{{% /steps %}}

## 明暗主题是怎么接上的

颜色切换落在 `<html>` 的两个属性上，缺一不可：

- `data-theme="light" | "dark"` —— 给 `tokens.css` 里的自定义属性用，整套配色只换这一个块；
- `class="light" | "dark"` —— 给 Chroma 的样式表用，它们生成时带了模式选择器，规则全部写成 `.light .chroma …` 和 `.dark .chroma …`。

两个值由 `layouts/_partials/head/theme-init.html` 里那段内联脚本在首次绘制之前写好（所以不会有闪白），随后由 `assets/js/modules/theme.js` 在读者点击切换时同步更新。两边共用同一个 localStorage 键 `hugo-scratch:theme`，改一边就得改另一边，否则刷新之后选择会跳回去。

代码块能跟着变色还有一个前置条件：`config/_default/hugo.toml` 里 `[markup.highlight]` 的 `noClasses = false`。它让 Chroma 输出类名而不是行内样式；如果改成 `true`，颜色会被写死在 HTML 里，上面那两套模式选择器就再也管不到代码块了。

## 重新生成 Chroma 样式表

`chroma-light.css` 和 `chroma-dark.css` 不是手写的，它们的第一行就记录了自己的生成命令：

```bash
hugo gen chromastyles --style=github --mode light --modeSelector \
  --classLight light --classDark dark > themes/hugo-scratch-theme/assets/css/chroma-light.css

hugo gen chromastyles --style=github-dark --mode dark --modeSelector \
  --classLight light --classDark dark > themes/hugo-scratch-theme/assets/css/chroma-dark.css
```

`--modeSelector` 把每条规则收进一个顶层类，`--classLight` / `--classDark` 决定那个类叫什么，必须与 `<html>` 上的 `class` 一致。想换配色，用另一个 `--style` 值重新生成即可；可以先把候选样式打到终端里看一眼再落盘。

## 前置条件与排查

Tailwind 那一段不是 Hugo 自己实现的，它调用 npm 装在站点根目录的 CLI，所以有几件事必须先成立，配置里也已经写好：

- 站点根目录执行过 `npm ci`（`package.json` 里有 `tailwindcss` 与 `@tailwindcss/cli`）；
- `[build.buildStats] enable = true`，并有把 `hugo_stats.json` 暴露成 `assets/notwatching/hugo_stats.json` 的模块挂载——Tailwind 只生成**渲染结果里真的出现过**的工具类，这份统计就是它的输入，`[build.cachebusters]` 里针对它的那条规则保证统计变化会让样式表重新生成；
- `[security.exec] allow` 里包含 `tailwindcss`，否则 Hugo 不会执行它；
- `layouts/_partials/head.html` 把 `head/css.html` 放在延迟模板里调用：统计文件要等所有页面渲染完才存在，内联调用会拿上一次构建的数据去编译，干净克隆上直接失败。

排查顺序和这张依赖表一致：页面完全没有颜色，先看 `public/css/bundle.min.<hash>.css` 有没有生成；只差 Tailwind 的工具类，先确认 `npm ci` 跑过、`hugo_stats.json` 存在；只有你新加的规则不生效，确认它写在 `custom.css` 或它导入的文件里；源文件和产物都对但浏览器还是旧的，指纹保证内容变化一定换文件名，所以先怀疑有东西缓存住了 HTML。
