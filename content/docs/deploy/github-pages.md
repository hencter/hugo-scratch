+++
title = '发布到 GitHub Pages'
linkTitle = 'GitHub Pages'
description = '用 GitHub Actions 构建并发布这个项目站点：子模块检出、固定 Hugo 版本、严格构建、Pages 源改到 Actions。'
date = 2026-03-02
weight = 10
difficulty = 'intermediate'
estimatedTime = 25
prerequisites = ['/docs/start/quick-start/', '/docs/configuration/site-config/']
outcomes = ['说清项目站点的 /hugo-scratch/ 前缀为什么不能去掉', '写出带子模块检出与严格构建的发布工作流', '把仓库的 Pages 源切成 GitHub Actions', '知道自定义域名要改哪些 DNS 记录和哪个配置项']
tags = ['部署']
+++

## 为什么 /hugo-scratch/ 这个前缀不能去掉

`config/_default/hugo.toml` 里的 `baseURL` 是 `https://hencter.github.io/hugo-scratch/`，这是一个 GitHub Pages **项目站点**：站点住在仓库名的子路径下。同时 `defaultContentLanguageInSubdir = false`，所以简体中文从 `/` 提供，英文从 `/en/` 提供，两个语言都在这个前缀之内。

Hugo 用 `baseURL` 生成所有绝对 URL：样式表与脚本的 `<link>`/`<script>`、每个页面的 canonical、Open Graph、`sitemap.xml` 和 RSS。把 `baseURL` 改成 `https://hencter.github.io/` 之后，页面本身仍然能打开，但每个资源都指向用户站点的根目录，结果是"有内容、没样式"。

{{< warning >}}
构建时的 `baseURL` 必须以 `/hugo-scratch/` 结尾。少一个尾斜杠或多一个子路径，症状都是资源 404，而不是构建报错。
{{< /warning >}}

## 仓库设置

在仓库的 **Settings → Pages** 里把 **Source** 改成 `GitHub Actions`，改动立即生效，没有保存按钮。

- 保持 `Deploy from a branch` 时，GitHub 会自己跑 Jekyll，你的工作流永远不会被用作发布来源；
- 这个源不需要 `gh-pages` 分支，也不需要提交 `public/`——产物由工作流上传；
- 从自定义 Actions 工作流发布时，GitHub 不创建 `CNAME` 文件，已存在的 `CNAME` 也会被忽略；自定义域名在 Pages 设置里填，不要往 `static/` 里放一个文件指望它生效；
- 第一次部署成功之后再勾 **Enforce HTTPS**，证书签发需要一点时间。

## 一份可用的工作流

`.github/workflows/pages.yml`：

```yaml
name: Build and deploy

env:
  HUGO_VERSION: 0.167.0

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v6

      - name: Install Hugo
        run: |
          curl -sfL --output-dir "${{ runner.temp }}" -O \
            "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir -p "$HOME/.local/hugo"
          tar -C "$HOME/.local/hugo" -xf "${{ runner.temp }}/hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "$HOME/.local/hugo" >> "$GITHUB_PATH"

      - name: Build
        run: |
          hugo build \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --ignoreCache \
            --panicOnWarning \
            --printPathWarnings \
            --printUnusedTemplates \
            --printI18nWarnings

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v5
        with:
          path: ./public

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

每一步在解决什么问题：

- **子模块**：`submodules: recursive` 把 `themes/hugo-scratch-theme` 一起检出。漏掉它的报错是 `found no layout file for "html" for kind "page"`，这个消息不会指向真正的原因，本地的修法是 `git submodule update --init --recursive`。
- **完整历史**：站点开了 `enableGitInfo = true`，`[frontmatter] lastmod = [':git', 'lastmod', 'date']`，所以浅克隆会让"最后更新"退回到 front matter 的值。
- **Hugo 版本**：下载的是**非 extended** 的 `hugo_<version>_linux-amd64.tar.gz`。主题在 `themes/hugo-scratch-theme/hugo.toml` 里声明 `[module.hugoVersion] extended = false`，本仓库也已在 0.167.0 的标准二进制上实机构建通过——Tailwind 集成本身不需要 extended。`min = '0.146.0'` 是下限，`env.HUGO_VERSION` 锁定的版本必须不低于它；这里用 0.167.0。
- **构建命令**：`hugo build` 与裸 `hugo` 等价，参数和仓库本地验证用的严格构建完全一致。`--panicOnWarning` 让第一个 WARNING 直接失败，所以线上不会带着"站内链接指向不存在的页面"这类问题发布。
- **发布**：`actions/upload-pages-artifact@v5` 把 `public/` 打包成 Pages 产物，`actions/deploy-pages@v5` 发布它；`permissions` 至少需要 `pages: write` 与 `id-token: write`，`environment: github-pages` 让部署地址出现在 Actions 的作业页面上。
- **一个不能省的安装步骤**：`npm ci`。样式表的 Tailwind 那一段调用的是站点根目录下的 Tailwind v4 CLI，它不是 Hugo 内置的，所以 CI 必须在校验构建之前先装依赖；`actions/setup-node` 加 `npm ci` 两步就够，锁文件让结果可重复。漏掉它时报的是找不到 `tailwindcss` 可执行文件，而不是某一页出错。
- **不需要的东西**：Dart Sass 和 Go 都不用装。装完依赖之后构建可以完全离线，所以上游示例里那些按文件存在与否条件安装工具链的步骤在这里是多余的；`config/production/hugo.toml` 也刻意没打开 `[minify]`，产物保持可读。

{{< note >}}
`--baseURL "${{ steps.pages.outputs.base_url }}/"` 是 `actions/configure-pages` 给出的地址，尾斜杠和子路径都由它带齐。如果你更希望只有一个真值来源，删掉这一行即可——配置里的 `baseURL` 表达的是同一个地址。
{{< /note >}}

## 从非 main 分支发布

工作流的**文件名不重要**，决定哪个文件运行的是它自己的 `on:` 触发器。要发布的分支不在默认分支上时：

- 把分支加进 `on.push.branches`（例如 `branches: [main, release]`），或者换成 `branches: [release]`；
- 保留 `workflow_dispatch:` 就能在 Actions 页面手动运行，`actions/checkout` 默认检出触发它的那个 ref；
- 想同时维护两条发布线，可以放两个工作流文件（例如 `pages.yml` 和 `pages-release.yml`），各自写自己的 `branches`；也可以只在一个文件里用 `if: github.ref == 'refs/heads/main'` 守住 `deploy` 作业；
- 两个发布工作流请共用同一个 `concurrency.group`（示例里是 `pages`），否则它们可能同时写 Pages，谁最后完成谁覆盖。

## 换一个托管：Netlify 与 Cloudflare Pages

两家的模型一样：连上仓库，给一条构建命令和一个输出目录。对这个仓库只有四处需要对齐——构建命令用上面那条严格命令（或裸 `hugo`）、输出目录填 `public/`、Hugo 版本通过 `HUGO_VERSION` 环境变量固定（同样 ≥ 0.146.0，非 extended 即可）、子模块必须在平台克隆时被初始化。最后一条最容易踩：平台没有跑 `git submodule update --init --recursive` 时，症状和本地漏掉子模块一模一样。

还有一处方向相反的差异：Netlify 和 Cloudflare Pages 的项目默认服务在根路径，而 GitHub 项目站点靠 `/hugo-scratch/` 前缀。所以换平台时 `baseURL` 要改成对方的域名（通常是 `https://<项目名>.netlify.app/` 或 `https://<项目名>.pages.dev/`），继续留着 `/hugo-scratch/` 会让资源指向一个不存在的子路径。

## 自定义域名

1. 在 **Settings → Pages → Custom domain** 填域名并保存。前面说过，Actions 发布不会生成 `CNAME` 文件，也不需要它。
2. 配 DNS。apex 域名（`example.com`）加四条 A 记录：`185.199.108.153`、`185.199.109.153`、`185.199.110.153`、`185.199.111.153`；`www` 建一条 CNAME 指向 `hencter.github.io`，注意不要带仓库名。
3. 改 `config/_default/hugo.toml` 的 `baseURL` 为 `https://example.com/`。自定义域名服务在根路径，`/hugo-scratch/` 前缀必须去掉，否则会出现和第一节完全相反的资源 404。
4. DNS 生效后回到 Pages 设置勾上 **Enforce HTTPS**。

改完域名后重新跑一次严格构建，然后直接读 `public/sitemap.xml` 和任一页面的 `public/**/index.html`，确认 canonical 与 `og:url` 已经换成新域名——这一步比在浏览器里刷新更早发现问题。

## 参考

{{< docref href="host-and-deploy/host-on-github-pages/" title="Host on GitHub Pages" >}}
