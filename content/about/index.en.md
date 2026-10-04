+++
title = 'About'
linkTitle = 'About'
description = 'What this repository is, who it is for, and what it deliberately does not do.'
date = 2026-01-05
weight = 10
+++

## What this repository is

Hugo Scratch is a site that **actually exercises** the common Hugo feature set
rather than listing it. Every feature maps to a file that exists in the
repository: semantic HTML structure, a complete SEO head, bilingual navigation and
content, on-demand client-side search, a switchable light/dark theme, a set of
shortcodes and Markdown render hooks, and CSS/JS pipelines built entirely by Hugo.

It is documentation as well. Every filename, configuration key and template path
that appears in the prose exists in the repository, so "read it once" and "change
it yourself" are the same exercise.

The site starts from `hugo new site` and `hugo new theme`. The theme lives at
<https://github.com/hencter/hugo-scratch-theme> and is mounted as a submodule at
`themes/hugo-scratch-theme`; the content and configuration belong to
<https://github.com/hencter/hugo-scratch>. The public site is
<https://hencter.github.io/hugo-scratch/>. The current version is `v1.11.0`.

## Who it is for

Three kinds of reader:

1. **Someone building a first Hugo site.** [Quick start](/docs/start/) is the whole
   path from cloning to seeing a page, and
   [Directory structure](/docs/start/directory-structure/) explains why each
   directory is where it is.
2. **Someone who wants to borrow a working config.** Start from
   `config/_default/hugo.toml` in
   [Site configuration](/docs/configuration/site-config/); the
   [CSS pipeline](/docs/assets/css/) section gives both build pipelines in full.
3. **A machine reading the documentation.** Besides HTML, every page emits a
   Markdown twin and a `pages.json` entry, and difficulty, estimated time,
   prerequisites and outcomes live in front matter rather than in prose — so an
   agent can decide whether a page is worth following or only worth consulting
   without parsing HTML.

## Three things it deliberately does not do

**It loads no third-party scripts.** A page requests same-origin resources only:
its stylesheet, its script, `search.json`, and the images on the page itself. No
CDN, no font service, no comment widget. The YouTube shortcode renders a **link**
rather than an embedded player, so no third-party request happens until the reader
chooses to leave.

**It collects no analytics by default.**
`layouts/_partials/analytics.html` emits a tag only when `[params.analytics]`
carries a value **and** the build is a production build. Both values in the
repository are empty strings, so visits to this site as it stands are not recorded
by any analytics service. That is not laziness; it is what a default should look
like. Collecting something should require someone to write it down.

**One Node dependency, and only at install time.** `package.json` lists exactly two packages, `tailwindcss` and `@tailwindcss/cli`, restored once by `npm ci` from the lockfile; after that every build is offline. The design system is inlined by Hugo's own `css.Build` and the scripts are bundled by the built-in `js.Build` (esbuild), so there is no SCSS, no PostCSS plugin and no build-script config file here — `css.TailwindCSS` is an official Hugo integration, not a pipeline we assembled.

{{< tip >}}
To use the scaffold as a template, the shortest path is `hugo new content` with the
theme's own archetype:
`hugo new content blog/my-post/index.md --kind blog` writes front matter with
`date`, `authors`, `tags`, `categories` and `series` already in place.
{{< /tip >}}

## Licensing

The site's **prose** (the text and images under `content/`) is licensed
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/): reuse, adapt and sell
freely, with attribution.

The **theme and code** — the theme repository, `layouts/`, `assets/`, and the code
samples inside the prose — are MIT licensed. Take them, change them, ship them;
keep the copyright notice and nothing more is asked.

The exact terms are on the [License](/legal/license/) page, and what the site does
and does not request in the browser is on the [Privacy](/legal/privacy/) page. Both
are maintained alongside the implementation: change a template or a config key,
change those two pages in the same commit.
