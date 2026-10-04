+++
title = 'Render hooks'
linkTitle = 'Render hooks'
description = 'Seven render hooks each take over one kind of Markdown element. This page triggers every one of them and says what it adds.'
date = 2026-02-22
weight = 50
difficulty = 'advanced'
estimatedTime = 25
prerequisites = ['/docs/content/markdown/']
outcomes = ['Know the input and output of all seven render hooks', 'Add build-time validation for internal links and images', 'Understand why Markdown-notation shortcode headings reach the TOC']
tags = ['render hooks', 'templates']
+++

A render hook is the layer between Markdown and the final HTML: **one kind of element** is rendered by your template instead of by the renderer. The file name is fixed and so is the location:

{{< filetree >}}
themes/hugo-scratch-theme/layouts/_markup/
├── render-heading.html       h1–h6
├── render-image.html         ![alt](src "title")
├── render-link.html          [text](href "title")
├── render-codeblock.html     fenced code blocks
├── render-blockquote.html    quotes, including > [!NOTE]
├── render-table.html         tables
└── render-passthrough.html   $$…$$ / \(…\) maths
{{< /filetree >}}

The directory is **`_markup`** — not `_partials`, not `partials`. A template in the wrong place does not raise an error; it is simply never called. The page looks fine minus that one layer of processing.

## Headings: ids, attributes and anchors

The hook receives `.Level`, `.Text`, `.Anchor`, `.Attributes` and `.PlainText`. It does three things: keeps the id Hugo computed (unless the body supplied `{#custom}`), passes block attributes through to the element, and appends an anchor link that appears on hover.

### This subsection has a custom id {#hook-heading-id}

Written as `### This subsection {#hook-heading-id}`. The link in the table of contents points at the same id, because the contents and the hook read the same `.Fragments`.

## Images: dimensions, lazy loading and a missing-file warning

The hook calls `resolve-image.html`, which looks in the **page bundle**, then `assets/`, then `static/`. When it finds the file it emits the intrinsic width and height Hugo read from it — which is the whole reason the page does not jump while images load:

![The default social card](/images/og-default.png)

A Markdown title becomes a `<figcaption>`:

![The default social card](/images/og-default.png "Add a title and the image gains a caption")

Remote images are exempt from the build-time check (they cannot be verified offline), but a site-relative path that resolves to nothing makes the hook call `warnf`:

> [!WARNING]
> Combined with `--panicOnWarning`, a missing image fails the build. That is the point: a broken image looks like blank space in a preview and only becomes a broken picture for a reader after deployment.

## Links: internal validation, external `rel`

A site-relative link is resolved once, and a warning is emitted if it resolves to nothing — so renaming a page surfaces every inbound link on the **next build** instead of waiting for a reader to hit a 404:

- internal: [Markdown and extensions](/docs/content/markdown/)
- external: [Hugo documentation](https://gohugo.io/documentation/)

External links gain `rel="noopener"` and a class; the arrow is added by CSS, because the hook's job is to make "this is external" true in the HTML, and the arrow is a styling concern.

The check can be switched off (`validateInternalLinks = false`) for a site that links to paths which only exist after deployment.

## Code blocks: highlighting, a language label and a copy button

The core of the hook is one line:

```go-html-template
{{ $result := transform.HighlightCodeBlock . }}
```

It returns the highlighted HTML; the hook then adds its own `div.highlight` wrapper, the language label, an optional filename, and a copy button. The button is **server-rendered**, so it is on the page even if the JavaScript bundle fails to load — it is simply inert:

```toml {filename="config/_default/hugo.toml"}
[markup.highlight]
  noClasses = false
```

`noClasses = false` is the precondition for the whole thing following the light/dark switch: Chroma emits class names rather than inline colours, and only then can two stylesheets define those colours.

## Quotes: five alert words mapped onto four styles

The hook looks at `.AlertType` first. If it is set, the quote becomes a callout — sharing **the same partial** as the `{{</* note */>}}` shortcode, so a callout written in prose and one written as a shortcode cannot end up looking like two different components. If it is empty, the quote is passed through:

> The most expensive part of documentation is the traps nobody wrote down.

> [!IMPORTANT]
> GitHub has five alert words — NOTE / TIP / IMPORTANT / WARNING / CAUTION — and there are four styles. The mapping lives in the hook, because `T "important"` would be a missing translation. That is a concrete case of the rule that an arbitrary string should never be handed straight to `T` from a template.

## Tables: one wrapper so the scrolling stays inside the table

The hook wraps the table in `.table-wrap`. That looks redundant until you open a seven-column table on a phone: without it, what stretches is the whole page, and the body text, footer and navigation all slide sideways with it.

| Hook | Key fields on the context | Triggered on this page |
| --- | --- | :---: |
| `render-heading` | `Level`, `Text`, `Anchor`, `Attributes` | yes |
| `render-image` | `Destination`, `Text`, `Title`, `Page` | yes |
| `render-link` | `Destination`, `Text`, `Title`, `Page` | yes |
| `render-codeblock` | `Type`, `Inner`, `Options` | yes |
| `render-blockquote` | `Text`, `AlertType`, `AlertTitle` | yes |
| `render-table` | `THead`, `TBody`, `Attributes` | yes |
| `render-passthrough` | `Inner`, `Type` | see below |

{.no-wrap-first-col}

## Maths: one wrapper for a client-side renderer

The `passthrough` extension hands the delimiters through untouched; the hook's only job is to wrap them in a stable container:

$$ e^{i\pi} + 1 = 0 $$

Inline works the same way: \(\nabla \cdot \vec{E} = \rho / \varepsilon_0\). The theme does **not** load a renderer. Without one the reader sees the source, which is more honest than markup that looks broken, and it means a site that never writes maths never pays for the script.

## One consequence that is easy to miss

Markdown-notation shortcodes (`{{%/* tabs */%}}`, `{{%/* steps */%}}`) let headings inside them reach the table of contents because they run **before** the Markdown renderer: their output is parsed as Markdown blocks. Standard-notation shortcodes run after it, `.Inner` is unrendered text, and headings inside them never appear in `.Fragments`.

That difference decides the choice: **if the content has headings and you want them in the table of contents, it must be Markdown notation.**
