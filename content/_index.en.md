+++
title = 'Hugo Scratch'
description = 'A scratch site that uses the whole common Hugo feature set — and documents every one of those uses so you can keep talking to an agent about it.'
+++

## A Hugo site you can keep talking to

This repository was produced by `hugo new site` and `hugo new theme`. The **theme is its own repository**, wired in as a git submodule at `themes/hugo-scratch-theme`, so the site and the theme version independently and a theme upgrade never touches your content.

Clone it and it runs:

```bash
git clone --recurse-submodules https://github.com/hencter/hugo-scratch.git
cd hugo-scratch
hugo server
```

`--recurse-submodules` is not optional. Without it `themes/hugo-scratch-theme` is an empty directory and the site builds with no templates at all. The symptom is unmistakable: `found no layout file for "html" for kind "page"`.

## The rest of this page is not hand-written

The section cards, latest posts, tag cloud and agent entry points below are all derived from the content tree by the theme's templates:

- section cards come from `.Sections.ByWeight`, so adding a content directory adds a card;
- latest posts come from the blog section's `RegularPages`, newest first, limited by `[params.home] latestCount`;
- the tag cloud comes from `site.Taxonomies.tags.ByCount`;
- the machine-readable links come from the home page's output formats, so removing a format from the config removes its link — the two cannot drift.

To change this page, edit `content/_index.md`. To change those blocks, edit `themes/hugo-scratch-theme/layouts/home.html`. Content and layout are deliberately kept apart.

## Where to start reading

If you have ten minutes, three files explain the whole site:

1. `config/_default/hugo.toml` — every switch, commented.
2. `themes/hugo-scratch-theme/layouts/baseof.html` — the document contract: it defines the blocks, page kinds only fill in `main`.
3. `content/docs/content/front-matter.md` — the front-matter contract that the sidebar, breadcrumbs, prev/next links and `pages.json` all read.

To follow along, start at [Quick start](/docs/start/quick-start/). To see which Hugo features are actually used, read the [feature matrix](/docs/reference/feature-matrix/). To hand the work to an agent, read the [agent workflow](/docs/agents/workflow/).

## Three rules this site keeps

**The build does not use the network, but the stylesheet pipeline installs once.** Nothing in the build calls `resources.GetRemote` or reaches a remote address. The stylesheet is compiled in two parts: the theme's design system goes through Hugo's own `css.Build`, which inlines every `@import` into one file, and the Tailwind v4 part goes through the official `css.TailwindCSS` integration — it runs the CLI installed by `npm ci` at the site root and emits only the utilities that really appeared in the rendered output. The two parts are then joined, minified, fingerprinted and served with an `integrity` attribute. JavaScript is bundled by esbuild inside Hugo (`js.Build`). So the order is: clone, `npm ci`, and from then on every build can be completely offline. The one other optional dependency is the Python script that regenerates the brand images.

**No third-party scripts.** Pages request same-origin assets only. The search index is fetched when the reader first opens the dialog; the `youtube` shortcode deliberately renders a link rather than an iframe, so nothing is requested from a third party until the reader chooses to leave. Analytics is off by default and only ever runs in a production build.

**A warning is a failure.** The repository's strict build is:

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

`--panicOnWarning` stops the build on the first warning. A deprecated config key, a template nothing reaches, a site-relative link that resolves to no page, an image that exists nowhere — each fails the build on purpose. In a preview these look harmless; in production they are dead links and broken structured data.
