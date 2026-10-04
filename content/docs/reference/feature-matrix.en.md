+++
title = 'Feature matrix'
linkTitle = 'Feature matrix'
description = 'Every Hugo feature this site uses: where it lives, how it is verified, and which documentation page covers it.'
date = 2026-03-01
weight = 10
difficulty = 'intermediate'
estimatedTime = 20
prerequisites = ['/docs/start/directory-structure/']
outcomes = ['Locate the implementation of any feature from one table', 'Know the verification command for each feature', 'Tell verified claims apart from inferred ones']
tags = ['reference', 'features']
+++

This table is the **single entry point** for "what does this site actually use". Every row points at a real file or a real command, and the verification column describes what I ran and the artefact I saw — not what ought to happen.

Convention: paths in the implementation column are relative to the repository root, and every command in the verification column runs from the repository root.

## Skeleton and configuration

| Feature | Implementation | Verification |
| --- | --- | --- |
| `hugo new site` / `hugo new theme` scaffolding | repository root + `themes/hugo-scratch-theme/` | `hugo new theme <name>` produces this shape |
| Theme as its own repository, mounted as a submodule | `.gitmodules`, `themes/hugo-scratch-theme` | `git submodule status` |
| Layered config (theme → `_default` → environment) | `config/_default/*.toml`, `config/production/hugo.toml` | `hugo config` |
| Explicit module mounts, including `hugo_stats.json` | `[module]` in `config/_default/hugo.toml` | remove the `layouts` mount and the build finds no templates |
| Site-wide cascade (per-path sitemap weight) | `[[cascade]]` in the same file | `grep changefreq public/zh-cn/sitemap.xml` |
| Dates from git (`enableGitInfo` + `:git`) | `[frontmatter]` in `config/_default/hugo.toml` | the page's "updated" date equals that file's last commit |
| `buildStats` writing `hugo_stats.json` | `[build.buildStats]` | read `hugo_stats.json` after a build |

## Content management

| Feature | Implementation | Verification |
| --- | --- | --- |
| Branch / leaf / headless bundles | `content/docs/**`, `content/blog/*/index.md`; the headless bundle `content/snippets/build-gate/index.md` | `hugo list all`; the headless bundle produces **no** `public/snippets/` yet its text appears on the pages that include it |
| Page resources and image handling | `content/blog/hugo-pipes/cover.png` | the image carries `width`/`height` |
| Content adapter (pages from data) | `content/changelog/_content.gotmpl` + `data/changelog.toml` | `public/changelog/v1-0-0/index.html` exists |
| Three taxonomies | `[taxonomies]` = tag / category / series | `public/tags/`, `public/categories/`, `public/series/` |
| Front-matter contract | `themes/hugo-scratch-theme/archetypes/docs.md` | `hugo new content docs/x.md` uses it |
| Aliases and redirects | `aliases` in `content/docs/start/quick-start.md` (the page moved from `/docs/quick-start/`) | `public/docs/quick-start/index.html` is a `<meta http-equiv="refresh">` page; the alias path is relative to the site root and Hugo adds the language prefix itself |
| Pagination (2 per page, deliberately small) | `[pagination] pagerSize` | `public/blog/page/2/index.html` |
| Year-grouped listing | `groupByYear` in `content/blog/_index.md` | the blog list contains `<h2 id="year-2026">` |
| Draft / future / expired and the `notice` banner | `layouts/_partials/banner.html` | `notice = "…"` shows immediately; Hugo EXCLUDES draft, future-dated and expired pages, so those three banners need `hugo -D`, `--buildFuture` and `--buildExpired` respectively — and `hugo list drafts`, `hugo list future`, `hugo list expired` enumerate them without building |
| Prev/next within a section | `layouts/_partials/page-nav.html` | the previous/next links under the article |

## Templates and rendering

| Feature | Implementation | Verification |
| --- | --- | --- |
| `baseof` document contract with a `main` block | `themes/hugo-scratch-theme/layouts/baseof.html` | every page kind defines only `{{ define "main" }}` |
| Page-kind templates (home/page/section/taxonomy/term/404) | same directory | all six kinds have build output |
| A partial that returns a value | `layouts/_partials/resolve-image.html` | returns `{url,width,height,resource}` |
| A partial that returns layout decisions | `layouts/_partials/layout/flags.html` | returns `{sidebar,toc}`; `baseof` picks the grid |
| Inline partials (`define` inside a partial file) | `layouts/_partials/sidebar.html` and friends | the sidebar tree, menu, TOC and list rows all use it |
| Shortcodes, standard notation (`.Inner` is raw) | `layouts/_shortcodes/note.html` and friends | see [Shortcodes](/docs/content/shortcodes/) |
| Shortcodes, Markdown notation (`.Inner` is rendered) | `tabs.html` / `steps.html` / `columns.html` | same page |
| Language-aware documentation reference | `layouts/_shortcodes/docref.html` + the `[docs]` config | the Chinese page points at hugozh.cn and the English one at gohugo.io from **one** call site; switching language switches the reference |
| Fragment reuse via a headless bundle | `include.html` + `content/snippets/` | `public/snippets/` does not exist, yet the fragment's text appears on both pages that include it |
| Icon primitive, sized in one place | `layouts/_partials/icon.html` + the `.icon` rule | delete the `.icon` rule and a callout icon fills the entire callout |
| A shared `.Store` between shortcodes | `tab.html` writes, `tabs.html` reads the parent's | the tab buttons are server-rendered |
| Overriding a built-in shortcode | `figure.html`, `youtube.html` | the site's file of the same name wins |
| Seven render hooks | `layouts/_markup/render-*.html` | see [Render hooks](/docs/content/render-hooks/) |
| A table hook that wraps for scrolling | `render-table.html` | the rendered table is wrapped in `.table-wrap` |
| Build-time validation from a render hook | `render-link.html`, `render-image.html` | write a broken link and the build warns |
| `templates.Defer` for deferred rendering | the CSS call in `layouts/_partials/head.html` | Tailwind needs the finished `hugo_stats.json` |

## Asset pipeline: CSS and JavaScript

| Feature | Implementation | Verification |
| --- | --- | --- |
| Official Tailwind integration (`css.TailwindCSS`) | `layouts/_partials/head/css.html` + `assets/css/tailwind.css` | the output contains `.mt-6`, `.flex` and other utilities used once each |
| Tailwind scanned from the rendered result | `@source "hugo_stats.json"` | measured: every class this site uses also occurs in a file Tailwind auto-scans, so deleting the line yields a byte-identical bundle. What it actually covers is a class that exists **only** in the rendered output, assembled by interpolation, which auto-scanning cannot see |
| Layering (theme < components < utilities) | the `@layer` statement prepended by `head/css.html` | the first `@layer` in the output is the order statement |
| Preflight deliberately skipped | comments and imports in `assets/css/tailwind.css` | lists still have markers; the theme's own reset applies |
| Design system merged with `css.Build` (`@import`) | `assets/css/design-system.css` | the output is a single stylesheet |
| Site styles appended, not replacing the theme's | the site's `assets/css/custom.css` | change a radius and find it in the output |
| `resources.Concat` joining both parts | `head/css.html` | there is still exactly one `<link>` |
| `minify` + `fingerprint "sha384"` + SRI | `head/css.html`, `head/js.html` | the tag carries `integrity` and `crossorigin` |
| `js.Build` (bundled esbuild) over ES modules | `assets/js/main.js` + `modules/*.js` | one deferred `<script>` carries all behaviour |
| `@params` virtual module | the `params` option in `head/js.html` + `import * as params from '@params'` | each language gets its own hashed bundle |
| Flash-free colour themes | `head/theme-init.html` + `assets/css/chroma-*.css` | `<html>` carries both `data-theme` and the class |

## Output formats and machine-readable surfaces

| Feature | Implementation | Verification |
| --- | --- | --- |
| Custom media types and output formats, declared by the theme | `themes/hugo-scratch-theme/hugo.toml` | `hugo config` lists `outputformats` |
| Which pages emit which format, decided by the site | `[outputs]` in `config/_default/hugo.toml` | see the table below |
| A Markdown twin per page | `layouts/page.md`, `layouts/list.md` | `/docs/start/quick-start/index.md` |
| `llms.txt` | `layouts/home.llms.txt` | `/llms.txt` |
| Client-side search index | `layouts/home.search.json` | `/search.json` |
| Agent-facing page index | `layouts/home.pages.json` | `/pages.json` |
| Valid YAML front matter on the twins | `transform.Remarshal "yaml"` | the twin opens with parseable YAML |
| Custom sitemap with translation links and weights | `layouts/sitemap.xml` | `/zh-cn/sitemap.xml` contains `xhtml:link` |
| Generated, environment-aware robots.txt | `layouts/robots.txt` | a production build says `Allow: /` |
| Custom RSS, throttled by `[services.rss] limit` | `layouts/rss.xml` | `/index.xml` |

| Page kind | Output formats |
| --- | --- |
| home | `html` `rss` `md` `llms` `search` `pages` |
| section / taxonomy / term | `html` `rss` `md` |
| page | `html` `md` |

## SEO and semantic HTML

| Feature | Implementation | Verification |
| --- | --- | --- |
| Absolute canonical, and paged archives canonicalise to themselves | `layouts/_partials/head/meta.html` | read `public/blog/page/2/index.html` |
| `noindex` everywhere outside production | same file, via `hugo.IsProduction` | look at any page under `hugo server` |
| Open Graph + Twitter Card | `head/opengraph.html` | `og:image` is absolute and carries dimensions |
| JSON-LD `@graph` | `head/schema.html` | the script element holds an **object**, not a string |
| hreflang and `x-default` | `head/alternates.html` | translated pages carry three (zh-CN / en-US / x-default); a page with no translation still carries a self-referential hreflang — taxonomy terms whose zh and en names differ, and content-adapter pages that exist in the default language only, genuinely have nothing to pair with |
| Search-console verification, silent by default | `head/verification.html` | left empty, the tag is absent |
| Landmarks and a skip link | `layouts/baseof.html`, `header`/`footer` | `skip-link`, `aria-current`, `<time datetime>` |
| Print stylesheet | `assets/css/print.css` | in print preview the navigation disappears and external links gain their URL |
| Reduced-motion support | `assets/css/base.css` | the `prefers-reduced-motion` branch |

## Multilingual and i18n

| Feature | Implementation | Verification |
| --- | --- | --- |
| One content tree, `.en.md` pairing | `content/**/*.en.md` | 39 bilingual pairs, 100% paired; a paired page carries three hreflang values |
| Per-language menus | `config/_default/menus.zh-cn.toml`, `menus.en.toml` | the main menu differs by language |
| i18n catalogue with identical key sets | `themes/hugo-scratch-theme/i18n/*.toml` | `--printI18nWarnings` stays silent |
| Pluralised strings | `readingTime`, `pageCount` and friends | `T "key" <number>` |
| Per-language date format | `[<lang>.params]` in `config/_default/languages.toml` | Chinese pages read "2026 年 3 月 24 日" |
| A language switcher that never 404s | `layouts/_partials/lang-switcher.html` | an untranslated page falls back to that language's home |
| Per-language JS bundles (translations inside) | `params.i18n` in `head/js.html` | `public/js/` holds two hashes |

## Navigation and interaction, without third-party scripts

| Feature | Implementation | Verification |
| --- | --- | --- |
| Primary menu with active state | `layouts/_partials/menu.html` | the current item carries `aria-current="page"` |
| Section sidebar, expanding only the current path | `layouts/_partials/sidebar.html` | only the ancestors of the current page are open |
| On-this-page contents with scroll highlighting | `layouts/_partials/toc.html` + `modules/toc.js` | scrolling sets `aria-current` |
| Breadcrumbs | `layouts/_partials/breadcrumbs.html` | they agree with the BreadcrumbList JSON-LD |
| Client-side search, index fetched on first open | `modules/search.js` | the network panel shows `search.json` only after opening the dialog |
| Colour theme switch | `modules/theme.js` + `theme-toggle.html` | the choice survives a reload |
| Keyboard-navigable tabs | `modules/tabs.js` | arrow keys move between panels |
| Copy button | `modules/copy.js` + `render-codeblock.html` | the label changes to "Copied" |
| Back to top | `modules/back-to-top.js` | appears after one screen of scrolling |
| Mobile navigation, CSS-driven | `modules/nav.js` | with JavaScript off the menu is still present and usable |

## Build, verification and deployment

| Feature | Implementation | Verification |
| --- | --- | --- |
| Strict build where a warning is a failure | `AGENTS.md`, README | the command below |
| Isolated builds so parallel writers cannot collide | `--cacheDir` plus `-d` | see [Agent workflow](/docs/agents/workflow/) |
| CI: install, strict build, deploy | `.github/workflows/*.yml` | see [Deploying to GitHub Pages](/docs/deploy/github-pages/) |
| The build runs offline | no `resources.GetRemote`, no network call in the pipeline; the Tailwind CLI is installed by `npm ci` beforehand | after installing dependencies, run the strict build with the network off |

## How to verify the whole thing

One command, and only a zero exit counts:

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

It blocks five classes of problem at once: deprecated config keys or template methods (`--panicOnWarning`), two pages writing to the same target path (`--printPathWarnings`), templates nothing reaches (`--printUnusedTemplates`), missing translations (`--printI18nWarnings`), and the broken links and images the render hooks report.

The documentation also recommends the site audit before publishing, because it surfaces two classes of problem that are otherwise silent:

```bash
HUGO_MINIFY_TDEWOLFF_HTML_KEEPCOMMENTS=true HUGO_ENABLEMISSINGTRANSLATIONPLACEHOLDERS=true hugo --ignoreCache
grep -rn "MISSING_TRANSLATION" public/       # missing-translation placeholders
grep -rn "raw HTML omitted" public/           # HTML discarded because unsafe was off
```

A third search looks for Hugo's shortcode placeholder; its full spelling is H&#xfeff;AHAHUGOSHORTCODE. It can only be written that way here: the moment the full placeholder prefix appears in **content**, the build aborts with `illegal state in content; shortcode token missing end delim` — and the error is attributed to whichever page was rendering, not necessarily the page containing it. Written as an HTML entity, the rendered output does not contain the literal string either.

{{< warning >}}
All three searches can match the documentation itself — this page quotes `MISSING_TRANSLATION` and `raw HTML omitted`, and [Navigation](/docs/configuration/navigation/) and [Markdown](/docs/content/markdown/) each quote one of them too. The repository's CI therefore excludes two things: **the pages that document those strings**, by path, and **`public/search.json`** in full — by design it embeds every page body, so that explanatory text travels with it, while auditing the HTML and the Markdown twins already covers every page. An audit either makes both exclusions, or it fails on its own explanatory text the first time it runs.
{{< /warning >}}

Finally, the build output is the evidence that a page rendered at all: the directory under `public/` has to exist. A silent console does not mean a page was rendered.
