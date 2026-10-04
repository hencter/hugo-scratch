+++
title = 'Discovery'
linkTitle = 'Discovery'
description = 'robots.txt, the multilingual sitemap, canonical URLs and hreflang, the Markdown twin of every page, llms.txt, pages.json, search.json and site verification.'
date = 2026-02-25
weight = 30
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/', '/docs/seo/']
outcomes = ['Tell which of these files are generated and which are static', 'Explain how the multilingual sitemap and hreflang relate', 'Find and verify the three machine-readable entry points']
tags = ['SEO']
+++

The previous two pages were about getting a single page right; this one is about how anyone finds it. Every entry point is a build artifact, and none of them is a hand-maintained list — which is exactly why removing an output format also removes its file, its footer link and its sitemap entry, instead of leaving a reference to an address that no longer exists.

## robots.txt is generated, not static

A static site usually drops `robots.txt` into `static/`. Not here: `layouts/robots.txt` generates it at build time, with `enableRobotsTXT = true` in `config/_default/hugo.toml` switching the behaviour on. The two branches differ sharply:

- A production build emits `User-agent: *` with `Allow: /`, plus a `Sitemap:` line. That line must be an **absolute URL**, produced by `"sitemap.xml" | absURL`; the robots.txt specification requires it, and crawlers ignore a relative one without complaint.
- A non-production build emits `Disallow: /` and names the environment in a comment. A preview deployment therefore has no chance of being indexed and cannot compete with the real site for the same index.

The test is `hugo.IsProduction`, and a plain `hugo` command *is* production — as noted in [this chapter's overview](/docs/seo/), but the consequence is more immediate here: look at `robots.txt` under a local `hugo server` and you will always see the version that blocks every crawler.

## sitemap.xml: many languages, per-page weights

`layouts/sitemap.xml` overrides Hugo's built-in template, and differs in three ways. First, each `<url>` that has translations emits a set of `<xhtml:link rel="alternate" hreflang="…">` entries, which tells a search engine these are language variants of one page rather than duplicates of each other. Second, `<changefreq>` and `<priority>` come from **each page's own** `.Sitemap.ChangeFreq` and `.Sitemap.Priority`, which is what lets the `[[cascade]]` blocks in `config/_default/hugo.toml` actually land in the file: `/blog/**` is `daily` with `0.9`, `/legal/**` is `yearly` with `0.2`, and everything else uses the `[sitemap]` defaults of `weekly` and `0.5`. Third, the page collection is resolved defensively (`.Pages`, falling back to `.Data.Pages`) and then **asserted** to be non-empty — a silently empty sitemap is far worse than a failed build, so this one fails loudly.

A multilingual site also gains an index at the root: `/sitemap.xml` is a `<sitemapindex>` listing one sitemap per language, such as `/en/sitemap.xml`. A search engine reads the index and then each list, so adding a language requires no template change at all.

## Canonical URLs, hreflang and x-default

`layouts/_partials/head/meta.html` owns the canonical URL and uses `.Permalink` directly, so it is always absolute. Paginated archives are the easy-to-miss exception: from page 2 onwards the canonical points at the pager's own URL rather than page 1 — otherwise pages 2..n would each declare themselves duplicates of page 1 and quietly drop out of the index. A single page can override the value with `canonical` in front matter.

Translation links live in `layouts/_partials/head/alternates.html`, and the same partial emits a second kind of `rel="alternate"`: the sitemap, the RSS feed and each page's Markdown twin are all declared as alternative representations of the page. The hreflang loop walks `.AllTranslations` (which always includes the current language) and takes its value from `.Language.Locale`, so the tags read `zh-CN` and `en-US`.

`x-default` is chosen once, pointing at the language named by `params.seo.xDefaultLang` — `zh-cn` here. It must not be guessed from language order: add a language, or reorder the `weight` values in `languages.toml`, and a guess changes underneath you. `x-default` is the landing page for a visitor who matched no language, so it deserves to be a decision.

## A Markdown twin per page, plus three machine-readable entries

`themes/hugo-scratch-theme/hugo.toml` declares an `md` output format: media type `text/markdown`, `baseName = 'index'`, `isPlainText = true`, `isHTML = false`. The site's `[outputs]` (see [Site configuration](/docs/configuration/site-config/)) then decides which pages produce it — pages are `['html', 'md']`, and so are sections, taxonomies and terms. Append `index.md` to any page's URL and you have its Markdown version, for example `/docs/assets/css/index.md`.

The "Markdown source" link at the bottom of a page comes from `.OutputFormats.Get "md"` in `layouts/page.html` rather than from a hand-written address. It and the `rel="alternate"` link are two renderings of the same datum.

Beyond the HTML and its twin, the site publishes three files meant for machines, all of them output formats of the home page declared in `themes/hugo-scratch-theme/hugo.toml`:

- **`/llms.txt`** (the `llms` format, `notAlternative = true`): plain text that first says what the site is, then lists the documentation by section, the latest posts, the machine-readable files and every page. Its purpose is to explain everything in one request — useful to a language model or any client without an HTML parser.
- **`/pages.json`** (the `pages` format): one record per page carrying `role`, `title`, `url`, `markdown`, `tags`, `wordCount`, `readingTime`, translation links and the front matter fields `difficulty`, `estimatedTime`, `prerequisites` and `outcomes`. It answers "how should this page be used", so it reads the same fields as the information panel on the HTML page and the two cannot drift apart.
- **`/search.json`** (the `search` format): the client-side search index, fetched by `assets/js/modules/search.js` the first time a reader opens the dialog. Its URL reaches the script through `@params`; the reasoning is in [the JavaScript pipeline](/docs/assets/js/). Each record's `body` comes from `.RawContent` rather than the rendered page, because the rendered page also contains heading anchors, copy-button labels and code-block language tags — none of which the author wrote.

`layouts/_partials/footer.html` lists these files in the footer alongside the sitemap and the feed, and takes their addresses from `.OutputFormats` — remove an output format and its link goes with it.

## Site verification

Both of these are optional and neither affects the entry points above: register the site with Google Search Console or Bing Webmaster Tools, then prove you own it. The template `layouts/_partials/head/verification.html` reads `[params.verification]` from `config/_default/params.toml`: fill in `google` and it emits a `google-site-verification` meta tag, fill in `bing` and it emits `msvalidate.01`. When both are empty it emits **no tag at all**, so the default output carries no dead metadata. Both keys are commented out in the repository — uncomment and paste your own value to use them.

If you would rather not touch the configuration, the verification services also offer an HTML-file method: drop the downloaded file into `static/` and Hugo copies it to the site root unchanged.

## A one-pass checklist

```bash
hugo --ignoreCache
head -5 public/robots.txt
head -20 public/sitemap.xml
head -30 public/pages.json
```

Check four things in order: the `Sitemap:` line in `robots.txt` is absolute; the root `sitemap.xml` is an index and every language is listed in it; every entry in `pages.json` has both `url` and `markdown`; and `search.json` already contains the page you just wrote. Then open any page's source and confirm a canonical URL, paired hreflang links and a JSON-LD block. When all four pass, the site is discoverable, parseable and quotable by machines.
