+++
aliases = ['/docs/quick-start/']
title = 'Quick start'
linkTitle = 'Quick start'
description = 'Check your Hugo version, clone the repository with its submodule, start the dev server, and verify what you are looking at.'
date = 2026-01-10
weight = 10
difficulty = 'beginner'
estimatedTime = 12
prerequisites = ['/docs/start/']
outcomes = ['Run this site locally', 'Recognise the error a missing submodule produces', 'Know which page to read next']
tags = ['Hugo']
+++

This page does one thing: get this repository to render on your machine. It explains no design decisions — those belong to the chapters that follow. The whole job is three commands, but one of them is easy to copy wrong, so the reason it cannot be skipped is spelled out below.

## Before you start

You need a Hugo binary at **0.146 or newer**. That floor is the theme's own declaration: `[module.hugoVersion]` in `themes/hugo-scratch-theme/hugo.toml` says `min = '0.146.0'`. Below it Hugo prints a `Module "hugo-scratch-theme" is not compatible with this Hugo version` **warning** and then finishes the build, so a plain `hugo` will not stop you — this repository's strict build (`--panicOnWarning`) will. The version that built this page is {{< version >}}.

Beyond that you need `git`, a text editor, and one network round trip to install dependencies. The stylesheet is compiled in two parts: the theme's design system goes through Hugo's own `css.Build` (no Node needed), and the Tailwind v4 part goes through the official `css.TailwindCSS` integration, which runs the Tailwind CLI installed with npm **at the site root** — so `npm ci` comes first after cloning. The JavaScript side is always bundled by esbuild inside Hugo and needs no extra tooling.

```bash
hugo version
```

## Clone, with the submodule

The theme is not a checked-in directory. It is a git submodule attached at `themes/hugo-scratch-theme`.

{{% steps %}}
{{% step "Clone with submodules" %}}
```bash
git clone --recurse-submodules https://github.com/hencter/hugo-scratch.git
cd hugo-scratch
```
`--recurse-submodules` tells git to enter the submodule and fetch it as soon as the main repository is cloned. Without it, `themes/hugo-scratch-theme` is an empty directory.
{{% /step %}}
{{% step "Check that the submodule is really there" %}}
```bash
ls themes/hugo-scratch-theme
```
You should see entries such as `hugo.toml`, `layouts/` and `assets/`. An empty directory means the site is not ready yet.
{{% /step %}}
{{% step "Already cloned? Catch up once" %}}
From the repository root:
```bash
git submodule update --init --recursive
```
{{% /step %}}
{{% /steps %}}

{{< warning title="What a missing submodule looks like" >}}
With an empty theme directory Hugo will not say "the submodule is missing". It reports something like `found no layout file for "html" for kind "page"` for every page, or it builds a site with no styling and no navigation at all. When you see "no layout file", check whether `themes/hugo-scratch-theme` is empty before you go looking at templates.
{{< /warning >}}

## Install the stylesheet pipeline's dependency

One part of the CSS is compiled by Tailwind v4, and `css.TailwindCSS` runs the CLI installed with npm at the site root — it is not something Hugo ships:

```bash
npm ci
```

The versions are pinned in `package-lock.json`, which is why this is `npm ci` rather than `npm install`: it restores exactly what the lockfile records, and it is therefore repeatable. Skip this step and the build stops in the Tailwind part reporting that it cannot find the `tailwindcss` executable — note that the message names no page, so it is easily mistaken for a template problem.

## Start the development server

```bash
hugo server
```

The server listens on port 1313 by default; open the address it prints. It builds in memory, leaving `public/` for your own production builds, and it rebuilds on every file change and reloads the browser — so the whole loop is edit, save, look at the browser. No server restart.

{{< tip >}}
If another process already holds 1313, `hugo server` picks a different port and prints the new address, so reading the terminal beats memorising a port number. To pin one, use `hugo server --port 1414`.
{{< /tip >}}

## Verify

Once the site is open, confirm three things in this order. Each failure means something different:

- The home page shows a title, a navigation bar and a language switcher, properly styled. Plain unstyled text means the theme did not load; correct layout with wrong colours is a stylesheet problem and out of scope here.
- "Docs" in the header reaches the [docs](/docs/) landing page and the sidebar lists the chapters. The sidebar appears only for the sections named in `[params.nav] sidebarSections`, and `docs` is one of them.
- Open `content/docs/start/quick-start.md`, change a sentence and save. The browser should reload on its own. If live reload does not work, every later edit costs you a manual restart.

To see exactly which files Hugo treats as content, print the inventory with `hugo list all`. To check whether an edit affects the build, run a one-shot build with `hugo --logLevel warn`.

## Next

With the site running locally, read [Directory structure](/docs/start/directory-structure/) next: it marks the directories you must not edit by hand, `public/` and `resources/` among them. Then read [Configuration](/docs/configuration/) to understand why this site keeps no `hugo.toml` at its root and how the theme, `_default` and environment layers merge — every configuration key mentioned in later pages is defined there.

## The gate before delivery

{{< include "build-gate" >}}

## Reference

{{< docref "getting-started/"  >}}
