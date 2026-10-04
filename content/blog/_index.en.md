+++
title = 'Blog'
linkTitle = 'Blog'
description = 'Build notes for this site: the scaffold, Hugo Pipes and a bilingual build, one pitfall per post.'
weight = 20
groupByYear = true
+++

## How these notes are arranged

This section holds the build notes for the site itself. Each post covers one
concrete problem rather than listing features, and every filename that appears in
a code block exists in the repository under that name.

`content/blog/_index.md` carries `groupByYear = true` in its front matter.
`layouts/_partials/page-list.html` reads that value and switches to
`collections.GroupByPublishDate`, which is why the list below is a set of year
headings instead of one flat list. Delete the line and the year headings go with
it.

{{< note >}}
Every post uses the same front matter: `date`, `authors`, `tags`, `categories`,
`series`. The taxonomies are declared under `[taxonomies]` in
`config/_default/hugo.toml`, and `series` is the custom one — it threads this set
of posts into a reading order.
{{< /note >}}

## What grouping by year actually does

All a post has to get right is `date`. The year heading is computed at render
time from `.Date.Format "2006"` rather than written into the prose. That also
means dates are not casual edits: change a `date` to another year and the post
quietly moves to another group, with nothing anywhere reporting a problem.

The heading itself is an `##` heading and the template pins an anchor like
`id="year-2026"` to it, so linking to "the 2026 batch" from outside the section
works.

## What this series covers so far

Right now it is a four-part series called `Building a Hugo site from scratch`:

- [Why start from the scaffold](/blog/hello/): what `hugo new site` and `hugo new theme` really generate, and why the theme became its own repository.
- Building CSS and JS with Hugo Pipes: `css.Build` inlines the theme's design system, `css.TailwindCSS` runs the Tailwind v4 CLI, `js.Build` bundles with esbuild — and why only the Tailwind stage needs `npm ci`.
- Two languages for one site: the `.en.md` pairing rule, the language switcher's fallback, and how `hreflang` differs from `og:locale`.
- The remaining posts appear as the site grows — one post per feature, published when it works.

## Who these notes are written for

They assume you can write Markdown and that you have Hugo installed. The site
requires 0.146.0 or later[^1]; the current build
runs {{< version >}}. They do not assume you write Go templates: wherever a
template has to change, the post names the file and the edit.

## Using the notes as documentation

Besides HTML, every page emits a Markdown twin, and the "Markdown source" link at
the foot of a page points straight at it — so a post can be pasted into an editor
or another repository whole, instead of being scraped back out of rendered HTML.

> [!TIP]
> The search box on the home page indexes the text of the whole site. Searching
> for a concrete identifier such as "Pipes" or "`@params`" lands on a post faster
> than searching for "build".

{{< details title="Why there are no comments" >}}
Comments need a third-party service, a third-party script, and a privacy notice
to go with them. For now `layouts/_partials/comments.html` renders nothing but a
link to an issue in the repository: readers who want to discuss can follow it,
and readers who do not never download an extra byte for it.
{{< /details >}}

[^1]: [Installing Hugo](https://gohugo.io/installation/)
