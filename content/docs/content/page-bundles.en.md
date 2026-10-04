+++
title = 'Page bundles'
linkTitle = 'Page bundles'
description = 'Leaf bundles versus branch bundles, how page resources are found, and why an image is looked up in the bundle before assets/.'
date = 2026-02-20
weight = 30
difficulty = 'intermediate'
estimatedTime = 20
prerequisites = ['/docs/content/', '/docs/content/front-matter/']
outcomes = ['Decide whether a directory needs index.md or _index.md', 'Put an image in a bundle and reference it from Markdown', 'Read the three-step lookup in resolve-image.html']
tags = ['Hugo']
+++

In Hugo a directory is more than a drawer for files. A directory with an `index.md` becomes a single page, and everything else in it becomes that page's resources. A directory with an `_index.md` becomes a section, and the `.md` files inside it each become their own page. This page covers the difference, and the exact route the theme takes to find one image.

## The three kinds of bundle

| | Leaf bundle | Branch bundle |
| --- | --- | --- |
| Index file | `index.md` | `_index.md` |
| Page kind | `page` | `home`, `section`, `taxonomy`, `term` |
| Descendant pages | Not allowed | Allowed |
| Resources | Reached through `.Resources` | Reached through `.Resources`, minus descendant bundles |

The third row is the binding constraint: **a leaf bundle cannot contain another leaf bundle**. Add `content/blog/hugo-pipes/part-two/index.md` and it will not become a child page — it becomes an ordinary file resource.

The third kind is the headless bundle: the content and the resources are there, but the bundle is never published as a page of its own. It is defined with `headless` plus the `build` options in front matter:

```toml
+++
title = 'Image library'
headless = true
+++

[build]
  publishResources = true
```

`headless = true` keeps the page from producing HTML and removes it from lists, RSS and the sitemap, while `.Resources` stays reachable from other templates — a reference-only asset directory. `build.publishResources` decides whether the files inside are actually copied into `public/`. The key is `build`, not `_build`.

## A real leaf bundle

`content/blog/hugo-pipes/` is the leaf bundle in this site, and it holds exactly two things:

{{< filetree >}}
content/
└── blog/
    ├── _index.md
    ├── hello/
    │   └── index.md
    └── hugo-pipes/
        ├── index.md
        └── cover.png
{{< /filetree >}}

`index.md` is what makes the URL `/blog/hugo-pipes/`, and `cover.png` sits beside it, so it is a resource of that page and can be referenced by a relative path:

```markdown
![The Hugo Pipes build output](cover.png)
```

Note the spelling: `cover.png`, not `/cover.png` and not `/images/cover.png`. A relative path tells Hugo to look in this page's bundle first. Writing `/images/...` makes it a site-root path instead, which only resolves when a matching file exists in `assets/` or `static/`.

## How the theme finds an image

`layouts/_partials/resolve-image.html` is the single place that answers this, and the Markdown image hook (`_markup/render-image.html`), the `figure` shortcode and the Open Graph tags all call it. It asks three questions in a fixed order:

1. `$page.Resources.GetMatch $clean` — the current page's bundle first;
2. `resources.Get $clean` — then the site's `assets/`;
3. neither — assume `static/` and build the address with `absURL`.

Starting with the bundle matters because only bundle and asset resources expose a `MediaType` and their dimensions. `.Width` and `.Height` are what let the template emit real `width`/`height` attributes, which is what stops the layout from jumping as images load. Files in `static/` are copied verbatim, so Hugo never learns how big they are — that branch returns `width` and `height` of 0.

The same order explains why a file of the same name in the bundle wins over `assets/`: the bundle is closer to the page, and overriding a site-wide default is precisely what it is for. `resolve-image.html` returns `{url, width, height, resource}`, and `resource` coming back `false` is the tell that the answer came from `static/`.

{{< warning "Match on a name narrow enough to be unique" >}}
`Resources.GetMatch` takes a glob. `cover.png` matches `cover.png`; `*.png` matches the **first** PNG in the bundle. With several images in one bundle, avoid globs that broad — reordering the files would silently change which image is used.
{{< /warning >}}

## Branch bundle resources and cascades

Branch bundles hold resources too, but their `.Resources` **excludes files inside descendant bundles**. `layouts/_partials/head/schema.html` resolves the site-wide `params.images` through `resolve-image` with the home page as its context; `images/logo.svg` lives in `assets/`, so that lookup takes the second route.

The other use for a branch bundle is setting values across a whole subtree, which is what a site-level `cascade` is for. `config/_default/hugo.toml` carries two of them, and one of them gives every page under `/legal/**` a different sitemap frequency:

```toml
[[cascade]]
  [cascade.sitemap]
    changefreq = 'yearly'
    priority = 0.2
  [cascade.target]
    path = '/legal/**'
```

`cascade.target.path` is a glob that selects the pages, and every other key is merged into their front matter. Adding a shared `tags` value or a `notice` to a whole chapter belongs here rather than repeated per page — the repeated copies are the ones that eventually miss a page.

## When to split a bundle

When a page has a single companion image, `content/blog/hello/index.md` next to `cover.png` is enough. When an image is reused across pages, or needs processing — resizing, cropping, converting — put it in `assets/`. Resources in `assets/` are **published only when something references them**, whereas everything in `static/` is copied whether it is used or not. On an image-heavy site that difference is the size of the output directory.

## Reference

{{< docref "content-management/page-bundles/"  >}}
