+++
title = '静态资源管线'
linkTitle = '静态资源'
description = 'CSS 和 JavaScript 如何从 assets/ 变成浏览器里的一个文件：Hugo Pipes 构建、内容指纹、SRI，以及主题资源和站点资源的合并边界。'
weight = 50
+++

这一章只回答两个问题：样式表从哪里来，脚本从哪里来。两者走的是同一种管线形状，但每一步的参数不同，坏掉时的症状也完全不同——样式丢了是整页塌掉，脚本坏了往往只是一个按钮不响应，很容易被当成浏览器的问题。

## 这一节的两页

[CSS 构建管线](/docs/assets/css/)从 `themes/hugo-scratch-theme/assets/css/design-system.css` 出发，说明 `css.Build` 怎样把十条 `@import` 内联成设计系统、Tailwind v4 那一段怎样只生成真正被用到的工具类、站点的 `assets/css/custom.css` 在哪一步被拼进去、Chroma 的明暗样式表为什么必须交给 `hugo gen chromastyles` 生成，以及站点自己的规则究竟该写进哪个文件。

[JavaScript 管线](/docs/assets/js/)走另一条路径：`assets/js/main.js` 经 `js.Build` 打包成单个 IIFE，`@params` 这个虚拟模块把搜索索引地址和界面译文直接注进代码，而七个模块各自只负责一件事。读完它你能判断一个新交互该写进哪个模块，而不是继续往 `main.js` 里堆。

## 管线由谁编译

这里没有 PostCSS 配置文件，也没有 Sass：设计系统里的 `@import` 由 `css.Build`（Hugo 内建）内联，脚本由内建的 `js.Build` 打包，两者都能在非 extended 的二进制上跑——`themes/hugo-scratch-theme/hugo.toml` 的 `[module.hugoVersion]` 写着 `extended = false` 和 `min = '0.146.0'`。

唯一需要外部工具的是 Tailwind 那一段：`css.TailwindCSS` 调用 npm 装在站点根目录的 Tailwind v4 CLI，所以构建之前要先 `npm ci`，并且 `[build.buildStats] enable = true` 加上把 `hugo_stats.json` 暴露进资源树的那条挂载必须就位——Tailwind 只生成渲染结果里真的出现过的工具类，那份统计就是它的输入。这些都在 `config/_default/hugo.toml` 里，`head/css.html` 的注释也重复了一遍因为它们缺一不可。

## 主题资源和站点资源怎么合

`assets/` 是 Hugo 的资源目录。主题的 `assets/` 和站点的 `assets/` 在查找时表现为同一个命名空间，但合并只发生在**文件级**：同一个路径只会存在一份，站点的那份覆盖主题的那份，而覆盖是静默的。

{{< note >}}
这就是站点的样式改动写在 `assets/css/custom.css` 的原因。如果站点提供一个 `assets/css/design-system.css`，它会整体替换主题的样式表，构建不报任何错，页面看起来却像是主题坏了。脚本同理：要加模块，就在主题的 `assets/js/modules/` 旁边新建站点自己的同名文件并从入口引入；直接放一个 `assets/js/main.js` 会把主题的入口整个换掉。
{{< /note >}}

## 构建产物长什么样

一次构建之后，`public/` 里与这一节有关的东西只有三类，文件名都由 `resources.Concat` 的命名加上 `fingerprint` 的哈希决定：

```text
public/css/bundle.min.<hash>.css   ← 设计系统 + Tailwind，合成一份，带 SRI
public/js/main.<hash-a>.js         ← 中文界面的脚本（@params 里带着中文译文）
public/js/main.<hash-b>.js         ← 英文界面的脚本（@params 里带着英文译文）
```

CSS 只有一个文件，因为两段编译在最后被拼到一起；脚本有两个，因为界面译文进了代码，每种语言各编译一次。这两个事实合起来解释了一件事：改一行样式会让所有页面同时拿到一个新的哈希，而改一句界面文案只会让一种语言的脚本换名字，另一种的缓存继续有效。

想在产物里确认某个类名，直接搜 `public/css/bundle.min.<hash>.css` 就够了。搜不到通常不是管线丢了它，而是它压根没有出现在任何渲染结果里——Tailwind 只生成扫描到的工具类，这也是 `hugo_stats.json` 必须参与构建的原因。

## 怎么验证这一节的说法

改完样式或脚本之后先确认依赖装好，再做一次一次性构建：

```bash
npm ci
hugo --ignoreCache
ls public/css public/js
```

你应该看到 `public/css/bundle.min.<hash>.css` 一份，以及 `public/js/main.<hash>.js` 两份——界面译文进了脚本，所以两种语言各有自己的产物。文件名里的那串哈希就是内容指纹——内容变一次，文件名变一次，所以浏览器缓存不需要靠手写 `?v=` 失效。CSS 那份缺了，说明两段编译里有一段失败了，构建日志会指出是哪一段。

两类失败要分开看。构建期的失败一定是"文件或依赖不在"：`head/css.html` 会指名缺的是哪个资源，Tailwind 那一段还会因为没跑过 `npm ci` 或 `hugo_stats.json` 缺失而停下。运行期的"样式没生效"则多半与产物无关——指纹只保证内容变化一定换文件名，不保证你不会打开缓存里的上一份 HTML。

脚本那边同理：产物缺失由 `head/js.html` 的 `errorf` 指名，而"某个按钮不响应"通常是模板没渲染出它要挂钩的元素。先去 `public/` 里确认那个数据属性真的在 HTML 中出现，再回头读模块代码，顺序反了会浪费很多时间。
