+++
title = 'Directory structure'
linkTitle = 'Directory structure'
description = 'What each directory in the repository is for, and which ones are build output that you must not edit.'
date = 2026-01-10
weight = 20
difficulty = 'beginner'
estimatedTime = 10
prerequisites = ['/docs/start/quick-start/']
outcomes = ['Describe what every top-level directory does', 'Tell source directories apart from build output']
tags = ['Hugo']
+++

Before changing this site, spend two minutes on the directory layout. Almost everything here is yours to edit, but a small set of names is written by the build. Edit those and you either lose the change on the next build or leave the build reading a state that contradicts itself. Mixing the two categories up is the most common cause of wasted work on a fresh checkout.

## The repository at a glance

Below is the current tree. The indentation is the real nesting; the annotations say what each entry is for.

{{< filetree >}}
hugo-scratch/
├── archetypes/                 templates for new content
│   └── default.md              the skeleton `hugo new` uses
├── assets/                     files processed by Hugo Pipes
│   ├── css/custom.css          site styles layered over the theme
│   └── jsconfig.json           resolution config for js.Build
├── config/
│   ├── _default/               the site configuration proper
│   │   ├── hugo.toml           main configuration
│   │   ├── languages.toml      languages and locales
│   │   ├── menus.zh-cn.toml    Chinese menus
│   │   ├── menus.en.toml       English menus
│   │   └── params.toml         site params the templates read
│   └── production/
│       └── hugo.toml           layered in for production only
├── content/                    every page, organised by URL
│   ├── _index.md               the home page
│   ├── about/index.md          /about/
│   ├── blog/                   blog section and posts
│   ├── changelog/_index.md     /changelog/
│   ├── docs/                   the documentation chapter
│   │   ├── _index.md           /docs/
│   │   └── start/              the Start subsection
│   └── legal/                  legal section
├── data/                       structured data templates can read
├── i18n/                       translation catalogue for UI strings
├── layouts/                    the site's own template overrides
├── static/                     files copied to the output verbatim
├── themes/
│   └── hugo-scratch-theme/     the theme, a git submodule
├── .gitignore                  ignores build output
└── .hugo_build.lock            build lock, written by Hugo
{{< /filetree >}}

## What each directory is responsible for

| Directory | What lives there | Safe to edit |
| --- | --- | --- |
| `content/` | Every page. The nesting maps straight onto URLs: `docs/start/quick-start.md` becomes `/docs/start/quick-start/` | Yes — this is your daily working area |
| `config/_default/` | Site configuration, split into `hugo.toml`, `languages.toml`, `params.toml` and two menu files | Yes |
| `config/production/` | Configuration layered in only for production builds | Yes |
| `layouts/` | The site's own templates, overriding same-named files in the theme | Yes, though prefer editing the theme |
| `assets/` | CSS and JS that must go through Hugo Pipes | Yes |
| `static/` | Files copied to the output root unchanged, such as site icons | Yes |
| `data/` | Structured data templates read through `site.Data` | Yes |
| `i18n/` | Key/value translations for UI strings; the theme's `i18n/` fills the gaps | Yes |
| `archetypes/` | The skeletons `hugo new` starts a page from | Yes |
| `themes/` | The theme, attached as a git submodule. Editing here edits another repository | Fine for local experiments; commit in the theme repository |

That last row deserves a second look. `themes/hugo-scratch-theme` has its own `.git`, so a change made inside it belongs to the theme repository, not to this site. If you want to change a theme template for this site, the right place is this site's `layouts/`, which takes precedence over the theme and is committed alongside the content.

## Directories you must never edit

Four names below look like ordinary files and directories. They are not source. Hugo writes them, `.gitignore` excludes them, and hand-editing each one fails differently:

- **`public/`** — the output of a production build. By default `hugo` writes the entire site here, and its contents can be rewritten at the start of every build. Nothing you change here survives into the next build.
- **`resources/`** — Hugo's resource cache, holding processed images, concatenated CSS and JS, and similar intermediates. Its entire purpose is to save you recomputation; editing it by hand means at best that the next build overwrites you, and at worst that the cache no longer matches the source files.
- **`.hugo_build.lock`** — the lock file Hugo creates to stop two builds writing the same output directory at once. It belongs in `.gitignore`, must not be committed, and must not be edited by hand. If it is left behind after an interrupted build, deleting it is normally enough.
- **`hugo_stats.json`** — written on every build by `[build.buildStats] enable = true`; it lists the tags and class names this build actually used, and Tailwind reads it so that it emits only the utilities that really appeared. It is output as well: editing it edits a report that will be regenerated.
- **`node_modules/`, `package.json`, `package-lock.json`** — the Tailwind v4 CLI and library, restored from the lockfile by `npm ci`. They are not build output, but `node_modules/` is not committed.

There is a quick way to classify any unfamiliar directory: read `.gitignore`. What is ignored is output; what is not is source. This repository's `.gitignore` lists exactly those four entries plus editor and OS noise.

## Two habits for finding files

First, derive the file path from the URL. `/docs/start/quick-start/` is `content/docs/start/quick-start.md`, and the section landing page `/docs/` is `content/docs/_index.md`. `_index.md` and `index.md` are not interchangeable: the first is a branch node, the second is a leaf bundle, and one directory cannot hold both.

Second, ask Hugo for the content inventory instead of counting files in a tree:

```bash
hugo list all
```

It lists every content file Hugo recognises, with its path, date and kind. If a page you just created is missing from that list, the filename is usually wrong (renaming `_index.md` to `index.md`, say) or the page is still flagged as a draft.

With those two habits in place, you are ready for the next chapter, which shows how these configuration files merge into the single set of settings a build actually uses.
