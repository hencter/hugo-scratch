+++
title = 'Markdown and extensions'
linkTitle = 'Markdown'
description = 'Which Goldmark extensions this site enables, and what each configuration key looks like in the rendered result.'
date = 2026-02-16
weight = 30
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/content/front-matter/']
outcomes = ['Know which Markdown syntax is extended here', 'Read the markup section of config/_default/hugo.toml', 'Know what render hooks apply to']
tags = ['Markdown', 'Goldmark']
+++

Everything on this page is **live**: each section is the real rendered result on this site, not a screenshot. All the switches live in the `[markup]` section of `config/_default/hugo.toml`.

## Headings, anchors and the table of contents

`##` through `###` headings get an id automatically and a `#` anchor link that appears on hover. Both come from the render hook `render-heading.html`, not from Markdown. The table of contents on the right is built from `.Fragments.Headings` and covers `##` and `###` only, because `[markup.tableOfContents]` sets `startLevel = 2` and `endLevel = 3`.

Do not write a level-one heading in the body: the template already renders the `<h1>`, and a second one gives the page two competing top-level headings.

## Tables: alignment, attributes and horizontal scroll

Tables go through `render-table.html` and are wrapped in `.table-wrap`, so on a narrow screen the table scrolls horizontally instead of the whole page.

| Output format | Template file | Media type |
| --- | --- | --- |
| `md` | `layouts/page.md`, `layouts/list.md` | `text/markdown` |
| `llms` | `layouts/home.llms.txt` | `text/plain` |
| `search` | `layouts/home.search.json` | `application/json` |

Colons control alignment:

| Left | Centre | Right |
| :--- | :---: | ---: |
| a | b | c |
| a longer cell | a longer cell | 1 |

{.no-wrap-first-col}

That trailing `{.no-wrap-first-col}` line is a **block attribute** belonging to the table above; it is parsed because `[markup.goldmark.parser.attribute] block = true`. With the option off it becomes an ordinary line of text after the table — which is the price of turning it on: a bare pair of braces immediately after a block is significant.

## Quotes and GitHub-style alerts

An ordinary quote:

> The traps nobody wrote down — where the error message points at another file — are what actually costs the time.

A `> [!NOTE]` block is converted by `render-blockquote.html` into a callout built from **exactly the same markup** as the `{{</* note */>}}` shortcode:

> [!NOTE]
> Both entry points share one partial, so their appearance cannot drift apart.

> [!WARNING]
> Five alert words (NOTE / TIP / IMPORTANT / WARNING / CAUTION) map onto four callout styles. The mapping lives in the render hook, because `T "important"` would be a missing translation.

## Code blocks, filenames and the copy button

Fenced code goes through `render-codeblock.html`: it calls `transform.HighlightCodeBlock` for the highlighting, then adds the language label, an optional filename and a copy button. The filename comes from an option in the fence info string:

```go-html-template {filename="layouts/_partials/head/css.html"}
{{- with resources.Get "css/design-system.css" -}}
  {{- $design := . | css.Build (dict "minify" false) -}}
{{- end -}}
```

Inline code is written `` `css.TailwindCSS` `` and renders as `css.TailwindCSS`. An unknown language falls back to plain text instead of failing.

## Lists: tasks, definitions and nesting

Task lists render as a `<ul>` with checkboxes, and they are still lists — `list-style` comes from the theme's `base.css`, because Tailwind's preflight is deliberately skipped:

- [x] A render hook catches broken links
- [x] A render hook catches broken images
- [ ] Add an automatic `srcset` for images

Definition-list **terms** get ids automatically (`autoDefinitionTermID = true`):

Hugo
: A static site generator written in Go. This project needs 0.146 or later.

Goldmark
: Hugo's default Markdown renderer; everything under `[markup.goldmark]` configures it.

## Maths: passed through, not rendered

The `passthrough` extension hands the delimiters to a client-side renderer untouched:

$$ \int_{0}^{1} x^2 \, dx = \frac{1}{3} $$

Inline maths works the same way: \(a^2 + b^2 = c^2\). This theme deliberately does **not** load KaTeX or MathJax — that is a sizeable third-party script and a site that never writes maths should not pay for it. Without a renderer the reader sees the LaTeX source, which is more honest than markup that looks broken.

## Raw HTML, attributes and emoji

With `[markup.goldmark.renderer] unsafe` enabled, HTML in the body is preserved verbatim:

<div class="callout callout--tip">
  <p class="callout__title">Hand-written HTML</p>
  <div class="callout__body">This is HTML written directly in the body, with no shortcode involved and nothing escaped.</div>
</div>

That same option is what makes **Markdown-notation** shortcodes such as `{{%/* tabs */%}}` work: their output is re-parsed as Markdown, and with `unsafe` off the whole block is replaced by `<!-- raw HTML omitted -->`.

Inline attributes can be attached to a heading:

### A subsection with a custom id {#custom-anchor}

Written as `### A subsection {#custom-anchor}`; the render hook reads `.Attributes` to decide the final id. Emoji come from `enableEmoji = true` :smile:.

## Typographer substitutions

With `[markup.goldmark.extensions.typographer]` enabled, straight quotes become curly, `--` becomes – and `---` becomes —. If you ever need a literal straight quote or a run of hyphens, remember the substitution happens — which is why this repository spells out every substitution target in the config instead of leaving the block empty.

## Reference

{{< docref "content-management/syntax-highlighting/"  >}}
