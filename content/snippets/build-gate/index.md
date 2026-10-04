+++
title = '严格构建门禁'
description = '交付任何改动之前必须跑通的那一条命令。'
# headless = true makes this a headless bundle: Hugo renders its content but
# publishes nothing — no URL, no listing entry, no Markdown twin. It exists only
# to be pulled in by {{< include >}}. Remove the line and the fragment becomes an
# ordinary page at /snippets/build-gate/, which is how you can prove the
# difference in one build.
headless = true
+++

交付任何改动之前，先在仓库根目录跑这一条命令，**退出码必须是 0**：

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

它同时挡住五类问题：被弃用的配置键或模板方法（`--panicOnWarning`）、两个页面写到同一个目标路径、没人调用的模板、缺翻译，以及渲染钩子报出的断链与缺图。

产物才是证据：`public/` 里存在对应目录，才算那一页真的渲染了。控制台什么都没说，不代表页面生成了。
