# Shortcodes

A shortcode is a template invoked from content. The author writes `{{</* name */>}}`, Hugo extracts
it before Markdown runs, and the template's output is merged into the page.

## When someone asks for a shortcode

Work these in order; the notation decision shapes everything else.

1. **What is the output, and does it still need Markdown?** Markup that the Markdown renderer must
   process (headings, lists, links inside the block) → Markdown notation. Finished HTML → standard
   notation.
2. **Which arguments?** Named (`{{</* note type="tip" */>}}`) reads better in content; positional
   (`{{</* figure src alt width */>}}`) is shorter. `.IsNamedParams` tells the template which it got;
   `.Get` handles both.
3. **Does it wrap content?** Paired form needs `.Inner`; the self-closing form `/*/>` does not.
4. **Where does it live?** The site's own `layouts/_shortcodes/` for one site; a theme's copy for
   reuse. Subdirectories namespace the name (`media/audio.html` → `{{</* media/audio */>}}`).
5. **Write the template, call it from one page, build, and read the generated HTML.** Then document
   the call syntax next to the content that uses it.

## Location, naming, lookup

```text
layouts/_shortcodes/
├── note.html              →  {{</* note */>}}
└── media/audio.html       →  {{</* media/audio src=… */>}}
```

`layouts/_shortcodes/` is the current template system (0.146+); `layouts/shortcodes/` is legacy. A
theme's `_shortcodes` directory takes part in the same lookup, which is how a site overrides a single
shortcode from a theme. Hugo selects by name, output format and language, most specific first:
`foo.en.html` → `foo.html.html` → `foo.html` → `foo.html.en.html`
(<https://gohugo.io/templates/shortcode/#lookup-order>).

## Notation decides rendering order

Documented at <https://gohugo.io/templates/shortcode/#rendering-order>:

1. Markdown-notation shortcodes (`{{% … %}}`) execute **before** the Markdown renderer, in document
   order.
2. The Markdown renderer runs.
3. Standard-notation shortcodes (`{{< … >}}`) execute **after** it.

Consequences worth saying out loud:

- `.Inner` is the **raw** inner text under *both* notations; the difference is what happens to the
  shortcode's **output** afterwards (G27 has the `<pre>` dump that proves it).
- Markdown notation: the output is rendered by the page's Markdown renderer, so an inner heading
  reaches `.TableOfContents`; a template that emits `.Inner` at the top level therefore looks like
  it rendered the Markdown.
- Standard notation: the output is inserted as-is — pipe `.Inner` through `markdownify` /
  `RenderString` yourself, and inner headings never reach the table of contents.
- A **block-level wrapper defeats the post-render**: `<div class="stage">{{ .Inner }}</div>` opens a
  raw HTML block, so inner Markdown inside it stays literal unless a blank line ends the block
  first. This is the constraint that decides whether a "show the live result" wrapper can hold
  Markdown at all.
- A standard-notation call earlier in the document still runs *after* a Markdown-notation call later
  in it.

## Built-in shortcodes need no site template

`figure`, `details`, `highlight`, `param`, `ref`, `relref`, `qr`, `youtube`, `vimeo`, `instagram`
and `x` ship **inside the Hugo binary** as embedded templates. They resolve in any project with no
`layouts/_shortcodes/` entry, so G1's `template for shortcode "…" not found` is *not* what you get
for them — it fires for names the site never defined (upstream's custom `code-toggle`, `new-in`,
`include`, … are the usual suspects when translating someone else's docs).

Two of them are not free:

- **`x`** resolves its markup at build time with `resources.GetRemote`. Offline — or behind a host
  that fails the default `security.http` policy — it warns and renders nothing; a warning-strict
  build (`--panicOnWarning`) fails outright (G28).
- **`qr`** encodes locally at build time and publishes a `qr_<hash>.png` into the publish directory
  root, so it stays offline-safe; the hash changes with the payload, so never hand-write the URL.

`gist` was deprecated in 0.143.0 and **removed** in 0.156.0: content still calling it fails the
build.

## Showing a shortcode's real output in the documentation

Code fences teach the syntax; a reader still has to build the site to see the result. A wrapper
shortcode closes that loop — the docs page carries the live artifact next to the escaped call.
Verified pattern (`layouts/_shortcodes/demo.html`):

```go-html-template
{{- $label := .Get "label" | default "实际渲染效果" -}}
<figure class="demo">
  <figcaption class="demo-label">{{ $label }}</figcaption>
  <div class="demo-stage">{{ .Inner | safeHTML }}</div>
</figure>
```

Invariants, all forced by G27:

1. **Call it with standard notation** and put only shortcode calls or finished HTML inside. Nested
   shortcodes render first, so the parent receives their output as `.Inner` — safe. Markdown prose
   inside is *not* rendered (the wrapper is a block-level HTML element).
2. **Do not wrap a Markdown-notation demonstration in it** — that output still needs the page's
   Markdown renderer, which the wrapper blocks. Write those calls directly in the page body.
3. The escaped source (`{{</* … */>}}`) goes in a fence immediately above the wrapper, so the reader
   sees call and result side by side. Never leave that call unescaped.
4. Style the stage so the live artifact cannot be mistaken for a code sample — its own border and a
   label, not another fence.

The same idea covers render hooks: for constructs a hook intercepts (links, code blocks,
blockquotes), put the real Markdown in the body and describe which hook produced the HTML. Where the
site has **no** hook for a construct, say so — "this is Hugo's default rendering" is itself the
useful fact.

## Methods

`Get`, `Params`, `IsNamedParams`, `Inner`, `InnerDeindent`, `Parent`, `Name`, `Ordinal`, `Position`,
`Page`, `Site`, `Scratch`, `Store`, `Ref`, `RelRef`
(<https://gohugo.io/templates/shortcode/#methods>). Reach the current page with `.Page`, and page
resources through it: `{{ with .Page.Resources.Get (.Get "path") }}`.

## Nesting

Nested shortcodes render inside-out: each child runs first, and the parent receives the rendered
output of its children as `.Inner`. A child reaches its parent's parameters with `.Parent`. Inline
shortcodes cannot be nested.

## Verified example: a callout

`layouts/_shortcodes/note.html`:

```go-html-template
{{- $type := .Get "type" | default "note" -}}
{{- $title := .Get "title" -}}
<aside class="callout callout-{{ $type }}">
  {{- with $title }}<p class="callout-title">{{ . }}</p>{{ end -}}
  <div class="callout-body">{{ .Inner | markdownify }}</div>
</aside>
```

Content (standard notation):

```md
{{</* note type="warning" title="未转义不是单页问题" */}}
Hugo 在 Markdown 解析**之前**就提取短代码……
{{</* /note */>}}
```

Renders to `<aside class="callout callout-warning"><p class="callout-title">…</p><div
class="callout-body">…<strong>…</strong>…<code>…</code>…</div></aside>` — the inner Markdown is
rendered because the template calls `markdownify` itself.

**Observed:** verified on Hugo 0.167.0 in a project whose other templates use the classic paths
(`layouts/_default/`, `layouts/partials/`): `layouts/_shortcodes/` still resolves, so the two
conventions can coexist. Mixing them remains a smell — put shortcodes in `_shortcodes/` and be
consistent about the rest.

## Shortcodes versus render hooks

Use a **render hook** (`layouts/_markup/render-*.html`) to change how existing Markdown constructs —
headings, links, images, code blocks, tables, blockquotes — render everywhere on the site. Use a
**shortcode** when the content author opts in, block by block. A hook cannot be called from content;
a shortcode cannot intercept a Markdown construct. If the request is "every image in every page
should…", it is a hook.

## Security

Inline shortcodes (defined inside content) are off by default: Hugo's model trusts template and
configuration authors but not content authors. Turning on `security.enableInlineShortcodes` extends
that trust to anyone who can write content — recommend it only when the user owns that risk.

## Verification

```bash
hugo --ignoreCache --printUnusedTemplates    # a template nothing calls is listed here
```

Then read the generated page. A missing close, a wrong argument name, or a `warnf` from the template
shows up in the HTML, and an unmatched call is a build error, not a silent no-op. Examples of
shortcode syntax *inside content* must be escaped — see the iron rules in `SKILL.md`.

## Runnable examples: the template *is* the code sample

For reference pages whose examples call functions or methods, a stronger version of the same idea:
keep the example as a **real template**, display its source, and execute it — so the code shown and
the output shown cannot drift.

Three layers, one job each:

| Layer | Lives in | Holds |
| --- | --- | --- |
| Implementation | `layouts/partials/examples/<namespace>/<name>.html` | an ordinary template; context is `dict "args" … "page" …` |
| Declaration | page front matter, e.g. `[[params.examples]]` | `id` (= template path), `title`, `args`, optional `note` |
| Placement | one shortcode call in the body | where the panel appears |

The panel partial does both things with the same file:

```go-html-template
{{- $src := os.ReadFile (printf "layouts/partials/examples/%s.html" $id) -}}
{{ transform.Highlight (strings.TrimSpace $src) "go-html-template" }}
{{ partial (printf "examples/%s.html" $id) (dict "args" $args "page" $page) }}
```

Why the ceremony pays off:

- **Zero drift by construction** — the displayed source is read from disk and the output comes from
  running that same file; there is no second copy to update;
- **Examples are verified** — a typo fails the build instead of shipping a plausible lie; do not add
  "tolerant" error handling that hides it;
- **Contributors add examples through content**, not templates: write the template once, declare
  `id`/`title`/`args` in front matter, drop the call where it belongs;
- **Machines get the same facts** — give the Markdown output format its own partial reading the same
  declaration. A page body's `.md` export is built from `RawContent`, so a body shortcode will *not*
  appear there on its own (G24).

Two practical notes: mark the read string `safeHTML` when the partial is an html/template but its
consumer is a plain-text output format, or the code sample arrives full of `&#34;`; and guard the
lookup with `errorf` so a renamed template fails loudly instead of rendering an empty panel.
