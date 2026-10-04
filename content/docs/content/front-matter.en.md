+++
title = 'Front matter'
linkTitle = 'Front matter'
description = "The fields this theme actually reads, the trap in TOML tables, and why dates come from Git commits."
date = 2026-02-20
weight = 20
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/content/']
outcomes = ['Write front matter this theme can read', 'Let lastmod follow the Git history', 'Use the build options to control listing and rendering']
tags = ['Hugo']
+++

Front matter is whatever sits between the two `+++` lines at the top of a page. It carries the page's identity beyond its URL — title, dates, ordering — and it decides what the templates can read. This page covers only the fields this site actually uses, plus the handful of spellings that fail quietly.

## TOML, YAML or JSON

Hugo recognises all three, and the first line tells it which one it is looking at: `+++` is TOML, `---` is YAML, and a leading `{` is JSON:

```toml
+++
title = 'Quick start'
weight = 10
+++
```

```yaml
---
title: Quick start
weight: 10
---
```

```json
{ "title": "Quick start", "weight": 10 }
```

This repository is TOML throughout: every page under `content/`, every file in `config/_default/`, and `data/changelog.toml`. Seeing `+++` therefore means the same syntax applies. Only one of the three formats has a trap worth memorising — JSON and YAML have no tables, so they cannot run into it.

{{< warning "The TOML table trap" >}}
A bare key written below a table header joins that table. The comment at the top of `config/_default/hugo.toml` exists for exactly this reason:

```toml
[frontmatter]
  lastmod = [':git', 'lastmod', 'date']
theme = ['hugo-scratch-theme']   # wrong: this line became frontmatter.theme
```

The result is not a syntax error. `theme` silently stops working, the build still succeeds, and the only complaint is "found no layout file for kind". Scalars belong above the first `[table]`. The same trap exists inside a page:

```toml
+++
title = 'Index entry only'
+++

[build]
  render = 'never'
weight = 20   # wrong: this became build.weight, so the page keeps the default order
```
{{< /warning >}}

## The fields this theme reads

| Field | Type | Who reads it |
| --- | --- | --- |
| `title` | string | `<h1>`, `<title>`, Open Graph, `pages.json` |
| `linkTitle` | string | Sidebar, breadcrumbs, cards, previous/next |
| `description` | string | `<meta name="description">`, list summaries |
| `date` | date | `<time datetime>`, ordering, the starting point for `lastmod` |
| `lastmod` | date | The "Updated" line, `dateModified` in the structured data |
| `weight` | int | Sidebar and in-section ordering |
| `difficulty` | string | The facts panel in `layouts/_partials/facts.html` |
| `estimatedTime` | int | The same panel, shown in minutes |
| `prerequisites` | []string | Links in the panel; each value is a site path |
| `outcomes` | []string | The "What you will learn" list in the panel |
| `tags` / `categories` / `series` | []string | The three taxonomy pages |
| `images` | []string | `og:image` and the `image` property in JSON-LD |
| `notice` | string | The banner `layouts/_partials/banner.html` renders at the top |
| `toc` | bool | Set it to `false` and this page gets no table of contents |
| `sitemap` | table | `changefreq` / `priority`, read by `layouts/sitemap.xml` |

`linkTitle` is the one field that works without being written and works better with it: the sidebar prints it and falls back to `title`. For long titles, `linkTitle` decides how many lines the sidebar entry takes.

`weight` only has to be unique **inside its own section**. The pages in this chapter run 10, 20, 30 and so on, spaced by ten, so a new page inserted between two of them takes 15 and nothing after it has to be renumbered.

## Dates: fields, Git, and the `[frontmatter]` mapping

Two settings in `config/_default/hugo.toml` work together:

```toml
enableGitInfo = true

[frontmatter]
  lastmod = [':git', 'lastmod', 'date']
  date = ['date', ':git']
```

`enableGitInfo = true` makes Hugo read each page's `.GitInfo` out of the repository — commit hash, author, subject. The lists under `[frontmatter]` are **fallback chains**: when Hugo needs `lastmod` it asks Git first, then the page's own `lastmod`, then falls back to `date`.

The observed result is the clearest explanation. `content/legal/privacy.md` is dated 2026-01-05, and in the built site its `dateModified` is the time of the last commit that touched the file — which is also why the page grows an "Updated" line. `layouts/_partials/page-meta.html` renders that line only when `.Lastmod` and `.Date` fall on different days. In other words: do not hand-write `lastmod`. Edit the file, commit it, and the date moves.

{{< tip >}}
Guard dates with `.IsZero` rather than testing `.Date` with `with`: `.Date` is a `time.Time` struct, so it is always truthy and an undated page cheerfully prints `0001-01-01`. Every date in `page-meta.html` is wrapped in an `.IsZero` check.
{{< /tip >}}

## `build`: a page that exists only in the index

When a page should be listed but not rendered, or rendered but never listed, reach for the `build` table:

```toml
+++
title = 'Index entry only'
+++

[build]
  list = 'always'
  render = 'never'
  publishResources = false
```

`list` takes `always` / `local` / `never`, `render` takes `always` / `link` / `never`, and `publishResources` is the only boolean. The key is `build` — the older `_build` spelling is gone, and using it earns a deprecation warning that becomes a hard failure under `--panicOnWarning`.

## Aliases and cascades

`aliases` publishes redirect pages for old addresses, which is how a rename keeps its inbound links alive:

```toml
+++
title = 'Front matter'
aliases = ['/docs/writing/front-matter/', '/docs/front-matter/']
+++
```

To set a value across a group of pages, do not copy and paste it — use a site-wide `cascade`. `config/_default/hugo.toml` carries one for the blog and one for the legal section:

```toml
[[cascade]]
  [cascade.sitemap]
    changefreq = 'yearly'
    priority = 0.2
  [cascade.target]
    path = '/legal/**'
```

`cascade.target` selects the pages (`path`, `kind`, `lang` and friends) and every other key is merged into their front matter. In the built site `/legal/privacy/` carries `changefreq` `yearly` and `priority` `0.2` while every other page keeps the `weekly` / `0.5` from `[sitemap]`. That sentence is verifiable output, not a guess.

The older `cascade._target` spelling is deprecated as of 0.156; the current key is `cascade.target`.

After changing a page, run `hugo --ignoreCache` once and read the page in `public/`. Front matter mistakes almost always show up in the rendered output rather than in the build log.
