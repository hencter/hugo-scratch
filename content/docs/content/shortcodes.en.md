+++
title = 'Shortcodes'
linkTitle = 'Shortcodes'
description = 'Every shortcode this theme ships, each called for real on this page, with the notation it uses and why.'
date = 2026-02-18
weight = 40
difficulty = 'intermediate'
estimatedTime = 20
prerequisites = ['/docs/content/markdown/']
outcomes = ['Know each shortcode and its arguments', 'Decide between standard and Markdown notation', 'Override a theme shortcode from the site']
tags = ['shortcodes', 'templates']
+++

Shortcodes live in `layouts/_shortcodes/<name>.html`. A site's shortcode and the theme's share one lookup chain, and **the site's file wins** — that is how you override one shortcode without forking the theme.

## Two notations, and what they actually decide

| Notation | Written as | What `.Inner` is | Do its headings reach the TOC? |
| --- | --- | --- | --- |
| Standard | `{{</* name */>}}` | unrendered text; you call `markdownify` | no |
| Markdown | `{{%/* name */%}}` | HTML already rendered by Markdown | yes |

Both are called for real below. The rule is one sentence: **if the content will contain headings and you want them in the table of contents, it has to be Markdown notation.**

## Callouts: note / tip / warning / danger

All four names share `layouts/_partials/shortcodes/callout.html`, so their appearance cannot drift apart. Standard notation; the theme pipes the content through `markdownify`:

{{< note >}}
`note` is the neutral one. Use the `title` argument for a custom heading.
{{< /note >}}

{{< tip "Do this first" >}}
The heading on this callout comes from a **positional** argument. Writing it as the named argument `title` is exactly equivalent, but one call may not mix the two.
{{< /tip >}}

{{< warning >}}
In standard notation `.Inner` is **unrendered** text, which is why the partial calls `markdownify`. The trade-off is that headings inside it never reach the table of contents.
{{< /warning >}}

{{< danger >}}
Use this one for "this will break something". Its colour and icon come from `--color-danger` and follow the light/dark switch.
{{< /danger >}}

## Disclosure: details

A native `<details>`, no JavaScript:

{{< details title="Open for a note" >}}
The browser already implements the disclosure semantics; the shortcode only supplies the styling and the copy. `open="true"` starts it expanded — note that the argument has to be compared as a **string**, because in a template the text `false` is truthy.
{{< /details >}}

## Inline pieces: badge and kbd

Self-closing, so the arguments are the entire content: {{< badge "New" >}}, {{< badge text="Beta" tone="accent" >}}, and press {{< kbd "Ctrl" >}} + {{< kbd "K" >}} to open search.

Note the second badge: Hugo rejects a shortcode call that mixes a positional and a named parameter (`badge "Beta" tone="accent"`) with "cannot mix named and positional parameters". Use all named or all positional.

## A fact about the build: version

{{< version >}} — the shortcode reads `hugo.Version`. Prose that says "needs version X or later" goes stale; a value read at build time cannot.

## File trees: filetree

Standard notation, and the content is **not** passed through Markdown — indentation, `*` and `-` are exactly what a file tree is made of:

{{< filetree >}}
content/
├── _index.md              home page (zh-cn is the default language)
├── _index.en.md           English home page
├── docs/
│   ├── _index.md
│   └── content/
│       ├── _index.md
│       └── shortcodes.md  ← this page
└── changelog/
    ├── _index.md
    └── _content.gotmpl    content adapter: generates pages from data
{{< /filetree >}}

## Tabs: tabs + tab (Markdown notation)

The parent and child share one `.Store`. Each child appends its title to the **parent's** store, and nested shortcodes render inside-out — so by the time the parent runs, every tab has registered. That is why the buttons can be server-rendered.

{{% tabs %}}
{{% tab "First way" %}}
The content is rendered by Markdown first, so this can contain **bold**, lists and code:

- the group is `{{%/* tabs */%}}`
- a child is `{{%/* tab "Title" */%}}`
{{% /tab %}}
{{% tab "Second way" %}}
The parent can also be split out, for example to wrap the whole set in something else; the controller only depends on the `role="tabpanel"` panels.
{{% /tab %}}
{{% /tabs %}}

## Steps: steps + step (Markdown notation)

The output is a real `<ol>`: a screen reader announces "list of 3 items", and the numbers survive printing.

{{% steps %}}
{{% step "Install dependencies" %}}
Run `npm ci` at the site root — that is where the Tailwind CLI is installed from.
{{% /step %}}
{{% step "Check the statistics file" %}}
`hugo_stats.json` is produced by `[build.buildStats] enable = true`; Tailwind reads it to learn which classes are really used.
{{% /step %}}
{{% step "Build" %}}
`hugo --ignoreCache`, then read the artefact in `public/` instead of re-reading your own source.
{{% /step %}}
{{% /steps %}}

## Columns: columns + column (Markdown notation)

The column count is passed to CSS as a custom property, so collapsing on a narrow screen is a stylesheet decision rather than a template one:

{{% columns cols=2 %}}
{{% column %}}
**Left column.** The content is rendered by Markdown, then laid out by the shortcode.
{{% /column %}}
{{% column %}}
**Right column.** The `cols` argument only sets the `--columns` custom property.
{{% /column %}}
{{% /columns %}}

## Images: figure (overrides the built-in)

A site's shortcode takes precedence over the built-in of the same name, so `layouts/_shortcodes/figure.html` takes over. It shares `resolve-image.html` with the image render hook and the Open Graph tags, which is why a page-bundle image works in all three:

{{< figure src="/images/og-default.png" alt="The default social card" caption="Hugo reads the intrinsic width and height from the file and writes them onto the element, which is what stops the layout from jumping while the image loads." >}}

## Video: video

The shortcode checks that the file exists before emitting a player. If it does not, you get an explanation instead of a player that silently shows nothing:

{{< video src="/videos/example.mp4" poster="/images/og-default.png" caption="Once example.mp4 is in static/videos/, this becomes a real playable video element." >}}

## External video: youtube (overrides the built-in)

The built-in version inserts an `<iframe>` as the page loads, contacting YouTube and setting cookies before the reader asked for anything. This override renders a link instead:

{{< youtube id="dQw4w9WgXcQ" title="Example: a link to an external video" >}}

The host follows `[privacy.youtube] privacyEnhanced`, so the key documented by Hugo still governs what this emits.

## Diagrams: mermaid

With the runtime off it falls back **visibly** to a highlighted code block, rather than emitting an empty container nothing will ever render:

{{< mermaid >}}
flowchart LR
  A[content] --> B(hugo)
  B --> C{hugo_stats.json}
  C --> D[Tailwind CLI]
  D --> E[public/css]
{{< /mermaid >}}

To render it for real, switch `[params.mermaid] enabled` on and load the mermaid runtime yourself — a sizeable third-party script is a decision for the site, not a theme default.

## Table of contents: toc

Prints the `.TableOfContents` Hugo generated for **this** page. It is a second implementation next to the one on the right, which makes the comparison useful:

{{< toc >}}

The right-hand contents are rebuilt from `.Fragments.Headings` by `layouts/_partials/toc.html`, which is what lets it carry `aria-current` and the scroll highlight; `.TableOfContents` returns a finished `<nav>` string — fast, but you do not control the markup. Both take their depth from `[markup.tableOfContents]`'s `startLevel` / `endLevel`, so they can legitimately disagree. That is configuration, not a bug.

## Changelog: changelog

Renders the release list straight from `data/changelog.toml`. The same data is expanded into one page per release under `/changelog/` by a content adapter:

{{< changelog >}}
