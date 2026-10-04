+++
title = 'Output formats'
linkTitle = 'Output formats'
description = 'Why one home page also exists as index.md, llms.txt and pages.json, and who decides those file names.'
date = 2026-02-20
weight = 20
difficulty = 'advanced'
estimatedTime = 25
prerequisites = ['/docs/templates/', '/docs/templates/templates/']
outcomes = ['Explain the split between the theme declaring formats and the site choosing them', 'Name a template correctly for a new format', 'Read transform.Remarshal in the Markdown output']
tags = ['Hugo']
+++

A page does not have to produce HTML alone. This site gives its home page an extra Markdown file, a plain-text file and two JSON files, and gives every regular page a Markdown twin. This page explains who does that, and why one of the templates is called `home.llms.txt`.

## The theme declares formats; the site decides who emits them

`themes/hugo-scratch-theme/hugo.toml` declares a media type and four output formats:

```toml
[mediatypes.'text/markdown']
  suffixes = ['md']

[outputformats.md]
  mediaType      = 'text/markdown'
  baseName       = 'index'
  isPlainText    = true
  isHTML         = false
```

A theme's configuration may set only `params`, `menu`, `outputformats` and `mediatypes`, and `[outputs]` is not among them. That produces a clean split: **the theme decides what a format looks like; the site decides which pages emit it.** `config/_default/hugo.toml` says:

```toml
[outputs]
  home = ['html', 'rss', 'md', 'llms', 'search', 'pages']
  section = ['html', 'rss', 'md']
  taxonomy = ['html', 'rss', 'md']
  term = ['html', 'rss', 'md']
  page = ['html', 'md']
```

Remove a name from those lists and the file stops being generated — and because the links in `layouts/_partials/footer.html` come from `.OutputFormats`, the links disappear with it instead of turning into dead ends.

## How an output file gets its name

A non-HTML template is named `{page-kind template}.{format name}.{suffix}`. Looking for `home.llms.txt`, Hugo checks `layouts/home.llms.txt` first and falls back to `layouts/_default/home.llms.txt` and the other lookup locations. This theme's names map one-to-one onto the URLs:

| Template | Output | Note |
| --- | --- | --- |
| `layouts/home.llms.txt` | `/llms.txt` | `baseName = 'llms'` |
| `layouts/home.search.json` | `/search.json` | The client-side search index |
| `layouts/home.pages.json` | `/pages.json` | The machine-readable page inventory |
| `layouts/list.md` | `index.md` for list pages | Sections, taxonomies, terms |
| `layouts/page.md` | `index.md` for regular pages | One per page |
| `layouts/rss.xml` | `index.xml` | The `rss` format |
| `layouts/sitemap.xml` | `sitemap.xml` | Overrides Hugo's embedded template |
| `layouts/robots.txt` | `/robots.txt` | Enabled by `enableRobotsTXT` |

The `md` format's `baseName` is `index`, which is why the Markdown twin is `index.md` rather than `page.md`: it sits next to `index.html` in the same directory, so appending `index.md` to any URL returns that page in Markdown.

{{< note >}}
The theme's layout root uses the current template system — `baseof.html`, `home.html`, `page.html` — with no `_default/`. A list page's HTML comes from `layouts/section.html` while its Markdown twin comes from `layouts/list.md`: **one page, two outputs, two differently named templates.** That is the easiest thing to get wrong when you go looking.
{{< /note >}}

## `isPlainText` and `notAlternative`

`isPlainText = true` makes Hugo render the format with `text/template` instead of `html/template`, so nothing is HTML-escaped. Markdown and JSON both need it: without it, `#` and `<` come out as entities and the Markdown is simply broken.

`notAlternative = true` takes a format out of `.AlternativeOutputFormats`, the set `head/alternates.html` walks to emit `<link rel="alternate">`:

- `rss` and `md` leave it unset, so every page advertises `/index.md` and `/index.xml` as alternative representations;
- `llms`, `search` and `pages` set it, because they belong to the home page alone; forcing them into every page's `<head>` would be noise.

## The multilingual root sitemap is an index

This site produces two kinds of sitemap, and it is easy to notice only one of them:

- `layouts/sitemap.xml` overrides the embedded template and emits one `/zh-cn/sitemap.xml` and one `/en/sitemap.xml`, reading each page's `.Sitemap.ChangeFreq` and `.Sitemap.Priority` and joining translations with `xhtml:link`;
- a multilingual site also gets a root `/sitemap.xml`, and that one is a **sitemapindex**: it lists nothing but the addresses of the per-language sitemaps.

The observed root file looks like this:

```xml
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap><loc>https://example.com/zh-cn/sitemap.xml</loc></sitemap>
  <sitemap><loc>https://example.com/en/sitemap.xml</loc></sitemap>
</sitemapindex>
```

The `Sitemap:` line in `robots.txt` has to be an absolute URL, and a crawler that follows the index finds the complete inventory for every language.

## Front matter in the Markdown twins

`layouts/page.md` and `layouts/list.md` emit YAML front matter with `transform.Remarshal "yaml"`. A hand-written `key: "{{ .Title }}"` breaks the day a title contains a quote; assembling a `dict` and letting `Remarshal` serialise it always produces valid YAML:

```go-html-template
{{- $meta := dict "title" .Title "url" .Permalink "tags" $tags -}}
---
{{ $meta | transform.Remarshal "yaml" }}---
```

`page.md` also copies `difficulty`, `estimatedTime`, `prerequisites` and `outcomes` into that header, from the same values the on-page facts panel reads, and adds a `translations` list with the absolute URL of every language version.

## How to verify a change

After `hugo --ignoreCache`, read the artefacts rather than the templates:

- `public/index.md`, `public/llms.txt`, `public/search.json`, `public/pages.json`, `public/index.xml` and `public/sitemap.xml` should all exist;
- any `public/docs/.../index.md` must open with valid YAML, not with an escaped string;
- `public/sitemap.xml` is the index; `public/zh-cn/sitemap.xml` is the per-page list.

To add a format the order is: declare `outputformats` in the theme, attach it to a page kind in the site's `[outputs]`, then write a `{kind}.{name}.{suffix}` template. Miss any one of the three and no file appears.[^1]

[^1]: Upstream documentation: [Output formats](https://gohugo.io/configuration/output-formats/)
