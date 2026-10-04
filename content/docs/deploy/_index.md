+++
title = '部署'
linkTitle = '部署'
description = '把 public/ 交给静态托管，并让 baseURL 与站点的真实访问地址完全一致。'
weight = 70
+++

## 这一章处理的事

`hugo` 的全部产物只有一个目录：`public/`。部署因此不是"让站点上线"这种模糊的任务，而是两件具体的事——把 `public/` 交给一个静态托管，并保证构建时的 `baseURL` 和访问地址完全一致。托管平台可以换，`baseURL` 只有一种正确答案。

这个仓库的部署难点不在平台，而在 `baseURL` 与实际访问地址是否对齐。`config/_default/hugo.toml` 里的 `baseURL` 是 `https://scratch.hugozh.cn/`：站点用自定义域名从**根路径**发布，所以生成的绝对 URL 一律不带子路径。把它写成带子路径的地址——例如 GitHub Pages 项目站点的默认地址 `https://<用户名>.github.io/<仓库名>/`——页面本身还能打开，但样式表、脚本和站内链接会全部 404：症状是"有内容、没样式"，而不是构建报错。

第二个容易漏掉的前提是主题：`themes/hugo-scratch-theme` 是 git 子模块，指向 `https://github.com/hencter/hugo-scratch-theme`。任何构建环境都必须递归检出子模块，否则 Hugo 一个模板都找不到，报错是 `found no layout file for "html" for kind "page"`——它不会说"子模块没拉下来"。

## 交付前的自检

仓库用同一条命令验证构建，本地和 CI 没有第二套：

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

`--panicOnWarning` 让第一个 WARNING 直接失败，所以这条命令既是构建也是检查：重复的输出路径、没有任何模板引用到的模板、缺失的 i18n 键、指向不存在页面的站内链接，都会在这里暴露。代价是本地调试时太严格——去掉 `--panicOnWarning` 就只剩警告。

{{< note >}}
`--printPathWarnings` 报的是"两个页面写同一个输出路径"。这类问题不会让构建失败，只会让后一个覆盖前一个，线上表现为"某个页面的内容对不上"。
{{< /note >}}

## 本章内容

[发布到 GitHub Pages](/docs/deploy/github-pages/) 是主线：从仓库设置里的 Pages 源，到一份带子模块检出、固定 Hugo 版本、跑严格构建、用 `actions/upload-pages-artifact` 与 `actions/deploy-pages` 发布的工作流，再到从非 `main` 分支发布时该改哪里。同一页还给出 Netlify 与 Cloudflare Pages 的对应做法，以及自定义域名要动的 DNS 记录和 `baseURL`。

## 这一章不涉及的事

- 站点的资源管线（CSS/JS 由 Hugo Pipes 打包）在[资源](/docs/assets/)一章；
- 机器可读的产出（`/llms.txt`、`/pages.json`、`/search.json`）在[输出格式](/docs/templates/output-formats/)；
- 生产环境的开关属于 `config/production/hugo.toml`，见[站点配置](/docs/configuration/site-config/)。
