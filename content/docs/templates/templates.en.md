+++
title = 'Templates and lookup order'
linkTitle = 'Templates'
description = 'baseof as the contract, partials that return values, context and $, and the fact that and/or evaluate every argument.'
date = 2026-02-20
weight = 10
difficulty = 'advanced'
estimatedTime = 25
prerequisites = ['/docs/templates/', '/docs/start/directory-structure/']
outcomes = ['Trace a page from content to HTML', 'Write a partial that returns a value', 'Avoid the traps in and/or, default and nil']
tags = ['Hugo']
+++

This theme has no `layouts/_default/`. It uses the current template system: `baseof.html` defines the document skeleton, page-kind templates supply only a `main` block, partials live in `layouts/_partials/`, shortcodes in `layouts/_shortcodes/`, and render hooks in `layouts/_markup/`.

## Lookup order: specific first, generic last

A render answers two questions: **which output format** and **which template**. Looking for `page.html`, Hugo walks outwards from the most specific location:

1. `layouts/docs/content/page.html` — type + section
2. `layouts/docs/page.html` — type
3. `layouts/docs/content/single.html` — type + section
4. `layouts/docs/single.html` — type
5. `layouts/page.html`
6. `layouts/single.html`

`layouts/docs/page.html` wins over `layouts/page.html`, and that was measured in this sandbox: dropping a file containing nothing but `page.html` into `layouts/docs/` immediately took over `/docs/content/front-matter/`, while `/docs/content/` was unaffected — a section page looks for `section.html` and never enters this chain.

{{< note >}}
This theme's file names map onto the older spellings: `single.html` corresponds to `page.html`, `list.html` to `section.html` / `taxonomy.html` / `term.html`, and `index.html` to `home.html`. Both sets still work, but a project should keep only one — mix them and you will edit a file and see nothing change.
{{< /note >}}

## `baseof.html` is the contract; pages only fill `main`

`layouts/baseof.html` owns `<html>`, `<body>`, the header, the three-column grid and the footer, and leaves one gap in the middle:

```go-html-template
{{- $pageClass := "page--list" -}}
{{- if .IsPage }}{{ $pageClass = "page--reading" }}{{ end -}}
{{- if .IsHome }}{{ $pageClass = "page--wide" }}{{ end -}}
<main id="main" class="page {{ $pageClass }}" tabindex="-1">
  {{ block "main" . }}{{ end }}
</main>
```

Those three names answer "**why** may this page deviate from the reading measure", and only `page--wide` actually changes the width (`.page` in `layout.css` is the measure, centred, by default). The home page is a hero plus a card grid, so it is wide; every other kind — **sections, taxonomies and terms included** — is a list and prose meant to be read, so it keeps the measure. Index pages used to be wide too, which put 115-character lines in the 63rem middle track of a TOC layout and pushed the table of contents to the far edge.

`layouts/page.html`, `layouts/section.html` and `layouts/home.html` each contain nothing but `{{ define "main" }}…{{ end }}`. The contract also leaves two blocks for a site to use — `head-extra` (in `layouts/_partials/head.html`) and `scripts` (at the bottom of `baseof.html`) — so a site can add tags or scripts without overriding `baseof.html` itself:

```go-html-template
{{ define "head-extra" }}<link rel="me" href="https://example.com/@me">{{ end }}
```

The `main` block in `page.html` is short: breadcrumbs, title, `banner`, `page-meta`, `facts`, `.Content`, then the taxonomies, previous/next navigation and comments.

## The full path of one render

Take this page, `/docs/templates/templates/`:

1. `baseof.html` calls `partial "layout/flags.html" .` and receives `{sidebar: true, toc: true}`, which adds the `layout--with-sidebar layout--with-toc` classes;
2. `partial "head.html" .` then calls `head/meta.html` (title, description, canonical), `head/alternates.html` (`<link rel="alternate">` for RSS and the Markdown twin, plus hreflang), `head/opengraph.html` and `head/schema.html`;
3. `page.html` defines `main` and renders the breadcrumbs and the facts panel;
4. `.Content` triggers the Markdown renderer, and every heading, code block, table, link and image passes through a hook in `layouts/_markup/`;
5. `partial "toc.html" .` rebuilds the table of contents from `.Fragments.Headings` — not from `.TableOfContents`, which only returns one pre-rendered HTML string;
6. `partial "scripts.html" .` runs, and the render lands in the empty `{{ block "scripts" . }}`.

{{< tip >}}
`.Content` is **lazy**: if a template never mentions it, the Markdown is not rendered and no render hook runs. When debugging a hook, check first that the page really calls `.Content`.
{{< /tip >}}

## Partials: print, or return

By default a partial writes its result into the output stream, as `partial "icon.html" "search"` does. A partial can also hand back a value with `return`, which the caller catches in a variable:

```go-html-template
{{ $flags := partial "layout/flags.html" . }}
{{ if $flags.toc }}…{{ end }}
```

`layout/flags.html` returns `{sidebar, toc}`, and `resolve-image.html` returns `{url, width, height, resource}`. The value is the point: `baseof.html`, `sidebar.html` and `toc.html` all ask one file for one answer instead of each computing it and disagreeing. There is a second kind that never touches the filesystem: the **inline partial**, declared as `{{ define "_partials/inline/…" }}` inside the file that uses it so that file can recurse into itself. `layouts/_partials/sidebar.html` and `toc.html` both build their trees this way — a name starting with `_partials/` is an inline partial; any other name is an ordinary template.

## Context, dictionaries and slices

- `.` is the current context: the page object in a page template, the current element inside a `range`;
- `$` is the context the template started with, so it always points at the page; inside a nested `range`, `$.Page` is the safe spelling;
- `dict` builds a dictionary and `slice` builds a slice, and the two are how a partial receives more than one value: `partial "card.html" (dict "page" .)`. A partial accepts exactly one context argument, so several values always travel as a `dict` — `layouts/_partials/page-list.html` is called as `(dict "pages" $pages "group" $group)`.

## Guards, and three real traps

```go-html-template
{{ with site.Params.tagline }}<p>{{ . }}</p>{{ end }}
{{ $show := index $ui "showSidebar" }}
{{ if eq $show nil }}{{ $show = true }}{{ end }}
```

{{< danger "Trap one: and / or evaluate every argument" >}}
`and $p $p.Title` still panics when `$p` is nil — `$p.Title` was evaluated before `and` ever saw it; `and` only receives the two results. To short-circuit, use nested `with`, or compute a safe string first:

```go-html-template
{{ $title := "" }}
{{ with $p }}{{ $title = .Title }}{{ end }}
{{ if and $p $title }}…{{ end }}
```
{{< /danger >}}

{{< warning "Trap two: default true cannot express an explicit false" >}}
`default` only takes over when the value is empty. A caller who writes `false` looks identical to a caller who wrote nothing, so the value becomes `true`. When the switch is about whether a key exists, reach for `isset`, or `index` plus an `eq ... nil` test — which is how `flags.html` reads `showSidebar`.
{{< /warning >}}

The third trap is dates: `.Date` is a `time.Time` struct, so `with` is always true for it, and "is there a date?" has to be asked as `.Date.IsZero`. Every date in `page-meta.html` carries that test.

## `.Store`: state passed between shortcodes

`.Store` is scratch storage attached to a page or a shortcode — write a key, read a key, and the value lives only for the current build. `layouts/_shortcodes/tabs.html` uses it to solve a real problem:

- each `tab` child appends `{id, title}` to the `tabTitles` key in `$parent.Store`;
- nested shortcodes render **inside-out**, so by the time the parent runs, every child title is already there;
- the parent reads the complete `tabTitles` and renders the tab buttons on the server in one pass — crawlers and readers without JavaScript see the whole component.

This theme uses `partialCached` for nothing: any claim that it caches anything here is false. When a partial needs to share state, write it to `.Store`; when you genuinely need caching, reach for `partialCached`, but measure first that it saves something.[^1]

[^1]: Upstream documentation: [Templates](https://gohugo.io/templates/introduction/)
