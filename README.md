# Hugo Scratch

一个把 Hugo 的常用特性**全部跑通**的脚手架站点：语义化 HTML、完整 SEO 头、双语导航、客户端搜索、明暗主题、短代码、渲染钩子，以及用 Hugo Pipes + 官方 Tailwind 集成搭起来的 CSS/JS 管线。

主题是**独立的仓库**，通过 git 子模块挂进来 —— 站点和主题可以各自演进、各自打标签，升级主题不会动到你的内容。

| | |
| --- | --- |
| 站点 | <https://hencter.github.io/hugo-scratch/> |
| 站点仓库 | <https://github.com/hencter/hugo-scratch> |
| 主题仓库 | <https://github.com/hencter/hugo-scratch-theme> |
| Hugo 版本 | 0.146 或更高（标准版即可，已用 0.167.0 的标准版与 extended 版实机构建验证） |
| 许可 | 代码与主题 MIT · 正文 CC BY 4.0，见 [许可说明](https://hencter.github.io/hugo-scratch/legal/license/) |

> **只想让 Agent 帮你改这个站点？** 根目录的 [AGENTS.md](AGENTS.md) 是唯一需要的文档：它写了运行方式、唯一那条必须通过的构建命令、内容字段契约、以及这个仓库真实踩过的坑。不需要再查别处。

## 跑起来

```bash
git clone --recurse-submodules https://github.com/hencter/hugo-scratch.git
cd hugo-scratch
npm ci                       # Tailwind v4 的 CLI，站点根目录这一份
hugo server                  # http://localhost:1313/hugo-scratch/
```

两个"不是可选项"的地方，以及它们失败时的样子：

- `--recurse-submodules`：少了它，`themes/hugo-scratch-theme` 是空目录，但报错长得不像主题问题——构建会停在配置阶段，报 `failed to create config: unknown output format "md" for kind "taxonomy"`（这些输出格式正是在主题里声明的）。目录被整个删掉时，报的是 `failed to load modules: module "hugo-scratch-theme" not found`。
- `npm ci`：样式表的 Tailwind 那一段调用的是 npm 装在本地的 CLI，不是 Hugo 内置的。少了它，报的是找不到 `tailwindcss` 可执行文件，同样不指向任何页面。

已经克隆过了，补一次就行：

```bash
git submodule update --init --recursive
npm ci
```

## 交付前的唯一一条门槛

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

退出码必须是 0。`--panicOnWarning` 让第一条 WARNING 直接失败，所以一个被弃用的配置键、一个没人调用的模板、一条解析不到页面的站内链接、一张找不到的图片，都会挡住发布——这些在预览里看着无害，上线后就是死链和失效的结构化数据。

产物才是证据：`public/` 里存在对应目录，才算那一页真的渲染了。`public/`、`resources/`、`.hugo_build.lock`、`hugo_stats.json` 都是**产物或状态**，不要手改。

## 里面有什么

这一页下面列的不是"计划支持"，每一条都有对应的真实文件与验证方式；完整清单见[特性总表](https://hencter.github.io/hugo-scratch/docs/reference/feature-matrix/)。

**内容组织**：分支包 / 叶子包 / 无头包（`content/snippets/` 只被 `{{< include >}}` 引用、从不发布）、页面资源与图片处理、内容适配器（从 `data/changelog.toml` 生成 `/changelog/` 下的一页页）、三种分类法、别名（旧地址 `/docs/quick-start/` 仍然重定向到新位置）、草稿与未来 / 过期页面（`hugo list drafts|future|expired` 可查，横幅由 `layouts/_partials/banner.html` 渲染）、按年分组的列表，以及刻意调小的分页（`pagerSize = 2`）。

**模板与渲染**：`baseof` 文档契约 + `main` 块、六种页面类型模板、会返回值的 partial、内联 partial、两套记法的短代码（`.Store` 在父子之间传递标题）、覆盖内置短代码、**七个渲染钩子**——其中两个会在构建期报出断链与缺图。

**资源管线**：`css.Build` 内联主题的设计系统、官方 `css.TailwindCSS` 只生成渲染结果里真正出现过的工具类、`resources.Concat` 合成一份、`minify` + `fingerprint "sha384"` + SRI；`js.Build`（内置 esbuild）打包 ES 模块，`@params` 把配置与译文注入前端，每种语言一份哈希。

**输出格式**：每个内容页都有 `index.md` 孪生页（合法 YAML 前置元数据；分页归档页除外，它们本身就是列表的第 N 页）、`/llms.txt`、`/pages.json`、`/search.json`、自定义 `sitemap.xml`（带语言互链与每页权重）、按环境生成的 `robots.txt`、限流的 RSS。

**SEO 与语义化**：绝对 canonical（翻页指向自身）、非生产环境一律 `noindex`、Open Graph 与 Twitter Card、JSON-LD `@graph`、hreflang 与 `x-default`、地标与跳转链接、`<time datetime>`、打印样式表、`prefers-reduced-motion`。

**双语**：单内容树 + `.en.md` 配对、每语言菜单、键集合必须一致的词条表、复数形式、每语言日期格式、不会 404 的语言切换器。

**没有第三方脚本**：页面只请求同源资源；搜索索引在第一次打开搜索框时才抓取；`youtube` 短代码只渲染链接而不是 iframe；统计默认关闭且只在生产构建生效。

## 目录长什么样

```
hugo-scratch/
├── AGENTS.md              给 Agent 的工作契约（也是本项目唯一需要的说明）
├── config/
│   ├── _default/          hugo.toml · languages.toml · params.toml · menus.<lang>.toml
│   └── production/        只在生产环境合并的覆盖
├── content/               一棵树，两种语言（.en.md 后缀）
├── data/changelog.toml    /changelog/ 内容适配器的数据源
├── assets/css/custom.css  站点自己的样式覆盖（主题保持不动）
└── themes/hugo-scratch-theme/   主题，独立仓库，git 子模块
```

仓库根目录就是站点根目录：`config/`、`content/`、`themes/` 都在这一层，不需要再进任何子目录。

## 部署

`.github/workflows/` 里有两个工作流：一个跑上面的严格构建，一个发布到 GitHub Pages。两者都必须是 `actions/checkout`（`submodules: recursive` + `fetch-depth: 0`）→ `actions/setup-node` + `npm ci` → 严格构建 → `upload-pages-artifact` + `deploy-pages`。仓库设置里把 Pages 的来源设为 GitHub Actions。

换域名只需要改两处：`config/_default/hugo.toml` 的 `baseURL`，以及 Pages 里的自定义域。项目站的 `/hugo-scratch/` 前缀来自 `baseURL`，改了它会跟着变。

## 许可

代码、主题与示例代码片段：MIT。`content/` 下的正文：CC BY 4.0。详见[许可说明](https://hencter.github.io/hugo-scratch/legal/license/)。

---

<sub>English abstract: a bilingual Hugo starter site that deliberately exercises the whole common feature set — semantic HTML, a complete SEO head, client-side search, light/dark themes, shortcodes, render hooks and a Hugo Pipes + official Tailwind v4 asset pipeline — with the theme as a separate repository wired in as a submodule. [AGENTS.md](AGENTS.md) is the complete working contract for an agent; no other document is required.</sub>
