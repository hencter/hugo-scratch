+++
title = 'Configuration'
linkTitle = 'Configuration'
description = 'Where the configuration lives, the order it merges in, and why TOML bit this project once.'
date = 2026-01-14
weight = 20
difficulty = 'beginner'
estimatedTime = 15
prerequisites = ['/docs/start/directory-structure/']
outcomes = ['Explain the precedence of theme, _default and environment configuration', 'Use hugo config to trace an effective value', 'Know what happens to a bare key written below a [table] header']
tags = ['Hugo', 'configuration']
+++

Hugo's configuration looks like the simplest part of the job: write a `hugo.toml`, put key/value pairs in it, done. But this site layers a theme, needs a production-only special case, and holds several unrelated concerns. Merge them into one file and changing a parameter means hunting through hundreds of lines for the table it belongs to. So the configuration is split — and splitting it buys a clear precedence chain plus one positional rule you have to respect.

## Why config/_default instead of a root hugo.toml

Hugo accepts two layouts: a single `hugo.toml` at the repository root, or a `config/` directory where each subdirectory is one configuration layer. This site picked the second, because it expresses three things a single root file cannot:

- **Separation of concerns.** The main configuration is `config/_default/hugo.toml`, the languages and locales are in `languages.toml` beside it, the params templates read are in `params.toml`, and the navigation is in `menus.zh-cn.toml` and `menus.en.toml`. Answering "where is the date format set" no longer means reading everything.
- **Somewhere for environment differences.** Extra settings for production builds go in `config/production/hugo.toml`. Local development is untouched by them, and no command-line flag is needed.
- **A file-level boundary between theme and site.** The theme's own `hugo.toml` is the first link in the chain and the site's configuration is the second, so which one overrides which is visible at a glance.

The cost is that "where is the configuration" stops having a single answer, so keep the merge order below in mind.

## Three configurations, one effective result

At build time Hugo stacks configuration in a fixed order, and later layers override earlier ones:

1. **The theme's `hugo.toml`** — `themes/hugo-scratch-theme/hugo.toml`. A theme configuration may contribute only `params`, `menu`, `outputformats` and `mediatypes`; every other top-level key in that file (a `baseURL`, say) is ignored. What lives here are the defaults that let a site build before it configures anything.
2. **`config/_default/`** — the site's configuration proper, with higher precedence than the theme. Every key in `config/_default/params.toml`, for example, beats the same-named key in the theme.
3. **`config/<environment>/`** — the environment layer, highest precedence of all. A plain `hugo` build runs in the production environment, so it reads `config/production/hugo.toml`. Set a different environment name and Hugo looks for a directory of that name; if there is none, nothing is layered in.

The files inside one directory are siblings, not nested levels: `hugo.toml`, `params.toml` and `languages.toml` do not override one another, they own different trees.

{{< note >}}
This project deliberately keeps no `hugo.toml` at the repository root. When both a root file and a `config/` directory exist, Hugo reads both, and which one wins depends on read order — exactly the kind of ambiguity that is expensive to debug. Pick one layout and keep it.
{{< /note >}}

## Three files, three jobs

- **`config/_default/hugo.toml`** — site identity and build behaviour: `baseURL`, `title`, `locale`, `defaultContentLanguage`, `enableGitInfo`, `timeZone`, `theme`, the `[module]` mounts, the `[frontmatter]` date sources, `[markup]`, `[taxonomies]`, `[pagination]`, `[outputs]`, `[cascade]` and more. To change how the site behaves, start here.
- **`config/_default/languages.toml`** — the `label`, `locale`, `weight` and `title` of the `zh-cn` and `en` languages, plus a `dateFormat` under each `[<lang>.params]`. It also records the pairing rule: `page.md` is the default language (`zh-cn`) and `page.en.md` is its English twin, matched by identical base name in the same directory, so the content does not need two parallel directory trees.
- **`config/_default/params.toml`** — the site-level params templates read: `description`, `author`, `repoURL`, a set of display switches under `[ui]`, `[nav] sidebarSections`, `[seo]` and so on. The theme ships a default for every key, so deleting one is not an error — it just falls back to the theme's value.

Menus are not in there. They live in `menus.zh-cn.toml` and `menus.en.toml` in the same directory; [Navigation](/docs/configuration/navigation/) explains why.

## The TOML positional trap

In TOML a bare key belongs to the nearest table header above it. Written like this:

```toml
[frontmatter]
  lastmod = [':git', 'lastmod', 'date']

theme = ['hugo-scratch-theme']
```

`theme` sits below `[frontmatter]`, so its real name is `frontmatter.theme`, not a top-level `theme`. Hugo reports no configuration error for this — it simply cannot find a theme and then complains, for every page, that it found no layout file. This project hit that trap for real: the site rendered with no theme at all while every error message pointed at templates.

The rule is easy to state: **every bare key goes above the first table header.** The top of both `hugo.toml` and `params.toml` carries a comment saying so, precisely so the next person reads it first. If you need to add a key to a table, put it inside that table and indent it to show the nesting.

{{< warning title="Read the build output after changing configuration" >}}
A configuration mistake usually shows up as "the page suddenly has no styling" or "that parameter does nothing", not as a clear error. After editing anything under `config/`, run a build and read its output — that finds the cause faster than refreshing a browser.
{{< /warning >}}

## Use hugo config to see effective values

Do not guess. `hugo config` prints the complete configuration this build actually uses, after the theme, `_default` and environment layers have merged, and including Hugo's own built-in defaults:

```bash
hugo config
```

The output is long, so filter it:

```bash
hugo config | grep -E 'theme|pagerSize|sidebarSections'
```

On PowerShell the equivalent is:

```powershell
hugo config | Select-String 'theme|pagerSize|sidebarSections'
```

The general rule is worth more than the example: any question of the form "what is this value actually set to" should be answered by the build tool, not by memory or by documentation. The same applies to flags — `hugo gen doc --dir <dir>` generates the full command-line reference for the Hugo you have installed, which is the only version that matters.

{{< tip >}}
`hugo config --printZero` includes zero-valued options (`false`, `0`, `""`) in the output, which is how you confirm a key is genuinely unset rather than merely equal to its default.
{{< /tip >}}
