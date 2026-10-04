+++
title = 'Docs'
linkTitle = 'Docs'
description = 'How this site is built, from the directory layout and configuration through content, templates and deployment.'
weight = 10
+++

This chapter is not a general Hugo tutorial. It is the manual for this repository: every page points at a file that actually exists, a configuration key that actually takes effect, or a command you will actually type. Finish a page and you should be able to open the file it names and change it, without hunting for a tutorial first.

## How the chapter is organised

The sections run in dependency order: get the site running, understand what it is made of, then descend one layer at a time. The order is **Start**, **Configuration**, **Content**, **Templates**, **Assets**, **SEO**, **Deploy**, **Agents**, **Reference**. Each section's `_index.md` is that section's table of contents, and the sidebar expands in the same order, so the navigation never disagrees with the reading order the prose recommends.

The sidebar expands only the branch you are currently inside and leaves the rest collapsed. A twenty-section manual with every branch open is a wall of links, which is harder to use than no navigation at all.

## What you need before reading

Three things:

- A Hugo binary at 0.146 or newer. The theme's `hugo.toml` declares `min = '0.146.0'` under `[module.hugoVersion]`, and anything older refuses to build at startup. The version that built this page is {{< version >}}.
- A copy of the repository with its submodules. The theme lives at `themes/hugo-scratch-theme` and is attached as a git submodule, so a plain clone leaves that directory empty.
- A text editor, plus one `npm ci`. The theme's design system and scripts go through Hugo's own `css.Build` and `js.Build` and need no Sass; only the Tailwind v4 stage runs the CLI that npm installs at the site root.

If you do not have a local copy yet, start with [Quick start](/docs/start/quick-start/). If the site already runs, go straight to [Directory structure](/docs/start/directory-structure/), which separates the directories you may edit from the ones that are build output.

## Every page carries its own "should I read this"

Besides a title and description, documentation pages declare four front-matter fields: `difficulty`, `estimatedTime`, `prerequisites` and `outcomes`. They are not decoration. `layouts/_partials/facts.html` renders them as the panel above the body, and the `pages.json` output format hands the same values to a machine. A reader can tell in three seconds whether a page is for them; an agent can tell whether to follow it without fetching the whole page.

{{< note >}}
The `difficulty`, `estimatedTime` and related fields apply to documentation pages. A **section index** (`_index.md`) does not need them — its job is to list its children.
{{< /note >}}

## What lives outside this chapter

The docs cover this repository. For why it is built this way and which traps it hit, read the [blog](/blog/) — those notes are a process record, not a reference. Site-level information (licence, privacy) is under [legal](/legal/), and version changes are in the [changelog](/changelog/).

When a page and a configuration file disagree, the file in the repository wins: prose can lag behind a commit, and the built page cannot.
