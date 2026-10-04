+++
title = 'Why start from the scaffold'
description = 'What hugo new site and hugo new theme really generate, and why the theme became its own repository.'
date = 2026-01-12
authors = ['Hencter Lew']
tags = ['Hugo', 'themes']
categories = ['engineering']
series = ['Building a Hugo site from scratch']
+++

## The scaffold gives you more than it looks like

`hugo new site` does one thing: it lays out empty directories and writes a minimal
`hugo.toml`. The step that actually gives you something is `hugo new theme`, and what
it emits is the **current template system** rather than the older layout many people
remember:

{{< filetree >}}
example/
├── archetypes/default.md   # template for hugo new content
├── assets/                 # files that need processing (Hugo Pipes entries)
├── content/  data/  i18n/  layouts/  static/
├── hugo.toml
└── themes/mytheme/
    ├── hugo.toml
    ├── archetypes/default.md
    ├── assets/
    │   ├── css/main.css
    │   ├── css/components/{header,footer}.css
    │   └── js/main.js
    ├── content/            # demo content: delete before shipping
    ├── layouts/
    │   ├── baseof.html
    │   ├── home.html  page.html  section.html
    │   ├── taxonomy.html  term.html
    │   └── _partials/
    │       ├── head.html  header.html  footer.html  menu.html  terms.html
    │       └── head/{css,js}.html
    └── static/favicon.ico
{{< /filetree >}}

Three things are easy to miss. First, templates sit at the root of `layouts/`, and
`baseof.html` hands each child template its slot through a `{{/* block */}}`. The
underscore in `_partials/` and `_shortcodes/` is a **reserved prefix**: get it wrong
and nothing errors, the template is simply never found.

Second, the theme's `hugo.toml` declares `[module.hugoVersion]`
(`min = '0.146.0'`) and ships three demo entries under `[[menus.main]]`. Menus are one
of the settings that **merge**, so leaving them in puts them in your navigation.

Third, the skeleton comes with `content/_index.md` and `content/posts/`. That is demo
content, not part of the template contract; this site's theme repository has no
`content/` directory at all.

## The two pipelines hiding in the skeleton

The most valuable files the scaffold writes are `layouts/_partials/head/css.html` and
`head/js.html`, because they already contain both Hugo Pipes pipelines. The CSS half
looks like this:

```go-html-template
{{- with resources.Get "css/main.css" }}
  {{- $opts := dict
    "minify" (cond hugo.IsDevelopment false true)
    "sourceMap" (cond hugo.IsDevelopment "linked" "none")
  }}
  {{- with . | css.Build $opts }}
    {{- with . | fingerprint }}
      <link rel="stylesheet" href="{{ .RelPermalink }}" integrity="{{ .Data.Integrity }}" crossorigin="anonymous">
```

`css.Build` inlines the `@import` statements in `main.css` into a single file, and
`cond hugo.IsDevelopment` keeps source maps in development while minifying only in
production. The JavaScript half differs by one function name: `js.Build` hands the
`import` graph to esbuild, which bundles it into one file. Of the two pipelines the scaffold handed over, only the stylesheet later gained an external dependency: the design system is still inlined by `css.Build` and the scripts are still bundled by `js.Build`, both inside the `hugo` binary, while the Tailwind v4 stage goes through `css.TailwindCSS`, which runs the CLI that `npm ci` installs at the site root.

## What this site changed on top of the skeleton

| Where | What the scaffold does | What this site does |
| --- | --- | --- |
| `head/css.html` | builds the theme's `main.css` only | also builds the project's `css/custom.css` and joins it with `resources.Concat` |
| `head/js.html` | calls `js.Build $opts` | passes a `params` value too, so esbuild exposes a `@params` virtual module |
| `fingerprint` | default `sha256` | `fingerprint "sha384"`, matching the `integrity` attribute |
| missing asset | `with` skips silently | an `errorf` fails the build |

That extra `params` value is one of the things that makes a bilingual site work, and
it gets a post of its own. The line worth remembering is the last one:
**`resources.Get` returns nil for a missing file and `with` silently skips the whole
block**, so the page renders unstyled while the build log says nothing at all.

## Why the theme is its own repository

Keeping the theme at `themes/hugo-scratch-theme` instead of under `layouts/` buys
three things:

1. **Changes stay traceable.** A commit that only touches styles or templates lives in
   the theme repository, so the theme's `main` is the theme's history. The site
   repository records nothing but a new parent commit pointer.
2. **A second site can reuse it.** The repository is
   <https://github.com/hencter/hugo-scratch-theme>; one line,
   `theme = ['hugo-scratch-theme']`, attaches it.
3. **Local development still reloads instantly.** `hugo server` reads that directory
   directly, so a CSS edit shows up immediately instead of waiting for a release.

The price is one extra step when cloning and when deploying: fetch the theme first,
then build. A single `git clone --recurse-submodules` of
`https://github.com/hencter/hugo-scratch.git` brings both down, which is what
[Quick start](/docs/start/) tells you to run. The theme contributes only `layouts/`,
`assets/`, `i18n/`, `static/` and its `params`; content and site configuration always
belong to the site repository — which is why the `baseURL` in the theme's own
`hugo.toml` has no effect.
