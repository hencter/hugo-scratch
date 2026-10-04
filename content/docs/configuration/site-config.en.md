+++
title = 'Site configuration'
linkTitle = 'Site configuration'
description = 'What each file under config/_default owns, how the three layers merge, and the TOML trap that silently costs the site its theme.'
date = 2026-01-14
weight = 10
difficulty = 'intermediate'
estimatedTime = 20
prerequisites = ['/docs/configuration/']
outcomes = ['Describe what each file under config/_default is responsible for', 'Change a setting in the right layer', 'Avoid putting a bare key below a [table] header']
tags = ['Hugo', 'configuration']
+++

Configuration is the layer you edit most often and the one that is hardest to debug when it goes wrong. It produces no page-level error: put a key in the wrong place and Hugo does not say so. It builds from the configuration as it understood it, and something else fails in a strange way. This page sets out the three merge layers, the boundary between the configuration files, and a real TOML trap this repository has already been bitten by.

## Why there is no root hugo.toml

There is no `hugo.toml` at the repository root. The configuration lives under `config/`, with `_default` holding the common part and a directory named after each environment holding the environment-specific part. Three things are bought with that: the main configuration, the languages, the params and the menus each become their own file, so changing one setting does not mean reading everything; environment differences have somewhere to live instead of being appended as a command-line flag; and the theme/site override relationship becomes a file-level fact rather than something you have to remember.

When a root file and a `config/` directory both exist, Hugo reads both. Since one layout is enough, there is no reason to keep two.

## Merge order: theme → _default → environment

At build time the configuration stacks in a fixed order, and later layers override earlier ones:

{{% steps %}}
{{% step "Theme configuration" %}}
`themes/hugo-scratch-theme/hugo.toml`. A theme configuration may contribute only `params`, `menu`, `outputformats` and `mediatypes`; any other top-level key in that file is ignored by Hugo. What it provides is the floor that lets a site build at all: `description`, `dateFormat = ':date_long'`, the display switches under `[params.ui]`, `[params.nav] sidebarSections = ['docs']` and so on.
{{% /step %}}
{{% step "Site defaults" %}}
Every `.toml` file under `config/_default/`. These beat same-named keys from the theme, and within `_default` the files do not override one another — `hugo.toml`, `languages.toml` and `params.toml` own different trees.
{{% /step %}}
{{% step "Environment configuration" %}}
`config/production/hugo.toml`. A plain `hugo` build runs in the production environment, so this layer is read and has the highest precedence. `hugo server` runs in the development environment by default, so a local preview shows the configuration without this layer — which is exactly why it exists.
{{% /step %}}
{{% /steps %}}

One rule of thumb decides which layer a key belongs in: anything independent of the deployment environment goes in `_default`, anything that is only true for a real release goes in the environment directory. The site description and the navigation params are the former; an analytics script you only want live, or stricter output settings, are the latter.

## The four files and their jobs

**`config/_default/hugo.toml`** owns site identity and build behaviour. It declares `baseURL = 'https://scratch.hugozh.cn/'`, `title`, `locale = 'zh-CN'`, `defaultContentLanguage = 'zh-cn'`, `defaultContentLanguageInSubdir = false` (so Chinese is served from the root and English under `/en/`), `enableGitInfo`, `hasCJKLanguage`, `timeZone = 'Asia/Shanghai'` and `theme = ['hugo-scratch-theme']`. Below that come the `[module]` mounts, the `[frontmatter]` date sources, `[markup]`, `[taxonomies]`, `[pagination]`, `[outputs]`, `[cascade]` and the rest. `baseURL` is the one key here tied to the deployment address: on a custom domain served from the root it carries no subpath, while the default URL of a GitHub project site would require `/<repo>/`.

**`config/_default/languages.toml`** owns languages. The two entries, `zh-cn` and `en`, each declare `label`, `locale`, `weight` and `title`, and give an explicit `dateFormat` under `[<lang>.params]` rather than relying on a localisation token such as `:date_long` — the localisation data does not cover every language.

**`config/_default/params.toml`** owns the params templates read: `description`, `tagline`, `author`, `repoURL`, `images`, plus `[ui]` (`showBreadcrumbs`, `showTableOfContents`, `tocMinHeadings`, `showPrevNext` …), `[nav] sidebarSections`, `[home] latestCount`, `[seo]`, `[analytics]` and similar tables. The theme ships a default for every key, so entries here are overrides, not requirements.

**`config/_default/menus.zh-cn.toml` and `menus.en.toml`** own navigation. Menus are neither in `params.toml` nor translated through the i18n catalogue; [Navigation](/docs/configuration/navigation/) explains why.

## The classic mistake: a bare key under a table header

TOML says a bare key belongs to the nearest table header above it. This looks perfectly reasonable:

```toml
[frontmatter]
  lastmod = [':git', 'lastmod', 'date']

theme = ['hugo-scratch-theme']
```

In fact `theme`'s full name is `frontmatter.theme`. Hugo reports no configuration error: it cannot find a theme and emits something like `found no layout file for "html" for kind "page"` for every page, which reads as a broken template. This repository was bitten by exactly that block — the site rendered with no theme at all while every error pointed at templates.

There is one way to avoid it, and it has to become a habit: **every bare key goes above the first `[table]` header**. Both `hugo.toml` and `params.toml` carry a comment at the top saying so, precisely so the next person reads it first. When you are adding a key to a table, write it inside that table and indent it, not at the end of the file.

## Use hugo config to trace effective values

Once the configuration has a shape, the remaining job is confirming that the layer you edited is the one in effect. Do not guess — ask the build tool:

```bash
hugo config
```

That prints the complete configuration after the theme, `_default` and environment layers have merged on top of Hugo's built-in defaults. The output is long, so it is normally filtered:

```bash
hugo config | grep -E 'theme|pagerSize|sidebarSections'
```

The PowerShell equivalent is `hugo config | Select-String 'theme|pagerSize|sidebarSections'`. To see the module mounts alone, `hugo config mounts` prints the result of `[module]` by itself — when templates or assets suddenly disappear, check that first, because it tells you immediately whether a mount is still there.

{{< note >}}
`[[module.mounts]]` under `[module]` deserves its own warning: declaring any mount replaces Hugo's default mounts wholesale. That is why this site lists all seven — `content`, `static`, `assets`, `layouts`, `i18n`, `data` and `archetypes`. Leave one out and that component quietly stops taking part in the build.
{{< /note >}}

After a configuration change the standard move is not to refresh the browser but to run a build and read its output:

```bash
hugo --logLevel warn
```

Warnings often explain more than errors do. To make the first warning fail the build outright, add `--panicOnWarning`, so a careless edit cannot sit unnoticed in the build log.[^1]

[^1]: Upstream documentation: [Configuration](https://gohugo.io/configuration/)
