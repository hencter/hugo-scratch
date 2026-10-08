# Site structure and front-matter conventions

A documentation-site layout that scales to hundreds of pages, composed from theme layers, and the
conventions that keep bulk edits from breaking the build. Adapt names, keep the shape.

**Labels used here.** This file records practice, not upstream statements: facts taken from the
public documentation are marked *documented* with a link; everything else is *observed* on a
working site.

**Two template systems, both valid.** The tree below uses the *classic* names
(`layouts/index.html`, `layouts/_default/…`, `layouts/partials/…`), which is what the site this
skill was distilled from uses. `hugo new theme` scaffolds the *current* system instead
(`layouts/baseof.html`, `home.html`, `page.html`, `section.html`, `layouts/_partials/`). Both work;
pick one per project, because mixing them at the same directory level is what produces
template-not-found surprises.

## Project layout

```text
my-site/
├── hugo.toml                  # config: baseURL, locale, menus, [markup], theme = […]
├── README.md
├── archetypes/default.md      # six-field front-matter template for `hugo new content`
├── content/
│   ├── _index.md              # home
│   └── <section>/
│       ├── _index.md          # section landing page (order = its `weight`)
│       └── <page>.md
├── layouts/                   # CONSTRAINT LAYER — only what every theme must obey
│   └── _default/
│       └── baseof.html        # the blocks and the partial names a theme has to provide
└── themes/
    ├── base-theme/            # `hugo new theme base-theme`
    │   ├── hugo.toml          # [params] / [module.hugoVersion] only — nothing else is honoured
    │   ├── layouts/
    │   │   ├── index.html     # home
    │   │   ├── 404.html
    │   │   ├── robots.txt     # production-only allow + `Sitemap:` line
    │   │   ├── _default/
    │   │   │   ├── single.html
    │   │   │   └── list.html
    │   │   └── partials/
    │   │       ├── head.html  # SEO head: canonical, OG, Twitter, hreflang, feeds, CSS
    │   │       ├── schema.html# JSON-LD (hand the object to the template, not `jsonify`)
    │   │       ├── header.html
    │   │       ├── sidebar.html
    │   │       ├── toc.html
    │   │       ├── footer.html
    │   │       └── scripts.html
    │   └── assets/
    │       ├── css/main.css   # layout + dark mode via prefers-color-scheme
    │       ├── css/syntax.css # Chroma class names when noClasses = false
    │       └── js/scrollspy.js
    └── zh-overlay/            # `hugo new theme zh-overlay`, then trimmed to one concern
        ├── hugo.toml          # [params.cjk] enabled = true
        └── assets/css/cjk.css # CJK typography only
```

```toml
theme = ["zh-overlay", "base-theme"]   # left wins; the project itself wins over every theme
```

Rules for the layers (documented in <https://gohugo.io/hugo-modules/theme-components/>):

- Files in `layouts`, `static` and `archetypes` override at **file level** — a same-path file
  replaces the other theme's file instead of merging. Keep an overlay's CSS in its own file and
  link both, or its rules silently replace the base stylesheet.
- `i18n` and `data` merge deeply by key.
- Delete the demo `[menus]` entries a scaffolded theme ships: menus merge, so they would appear in
  your navigation.
- A theme may configure only `params`, `menu`, `outputformats`, `mediatypes`.

## Front-matter contract

```toml
+++
title = "页面标题"
linkTitle = "侧栏与上一篇/下一篇里显示的名字"
description = "一句话导语，同时用于 meta description 与列表页"
date = 2026-01-01
weight = 10
source = "https://upstream.example/page/"   # only for translated pages
+++
```

- `weight` orders the sidebar, the section listing, and prev/next inside the section. Unique per
  section; section `_index.md` weights order the sections themselves.
- `description` doubles as the visible lead paragraph — write it as prose, not keywords.
- Keep the same key order everywhere. A build is not the place to discover a missing key.

## Navigation patterns that scale

- **Sidebar**: iterate `site.Home.Sections.ByWeight`; expand only the section equal to
  `.CurrentSection`, otherwise a 20-section site renders a wall of links.
- **TOC**: `{{ with .TableOfContents }}` inside an `<aside>`; hide it below ~1180px so the
  three-column grid collapses cleanly.
- **Scroll highlighting** (optional): a small dependency-free script that marks the current
  heading link and keeps it visible; derive the trigger threshold from the header height and
  write the same value into `scroll-padding-top` so anchor jumps and highlighting agree.
- **Home page**: section cards with `N pages · first links · view all`, not every page listed.
- **Prev/next**: order siblings with `.CurrentSection.Pages.ByWeight` and index the current page,
  rather than trusting `.NextInSection`/`.PrevInSection` ordering.

## Multilingual

- One file per language (`about.zh.md`, `about.en.md`) or one content directory per language via
  `contentDir`; branch bundles use `_index.zh.md`.
- Keep `defaultContentLanguage` and `locale` distinct (*documented*): the former picks the default
  language key, the latter is the BCP-47 tag (`languageCode` was renamed to `locale` in 0.158.0).
- UI strings live in `i18n/<lang>.toml` and are read with `i18n`/`T`.

## Translations of upstream documentation

- Put the upstream URL in `source` and render it in the footer, so a reader can always verify.
- Mirror the upstream page set one-to-one (one page per upstream page) and keep a scope note for
  sections you deliberately skip.
- Expect upstream restructures: pages move between sections, and a page can degrade into a stub
  that points at a new location. Recheck structure when you refresh a batch.
- Do not translate legal text verbatim by machine; summarise per section and link upstream.
