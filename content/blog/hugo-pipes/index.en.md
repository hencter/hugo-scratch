+++
title = 'Building CSS and JS with Hugo Pipes'
description = 'The two pipelines in detail: css.Build inlining @import, js.Build bundling and injecting @params, and why only the Tailwind stage needs an npm-installed CLI.'
date = 2026-02-18
authors = ['Hencter Lew']
images = ['cover.png']
tags = ['Hugo', 'Hugo Pipes']
categories = ['engineering']
series = ['Building a Hugo site from scratch']
+++

## What each pipeline does

![Cover of the Hugo Pipes post](cover.png "Processing with Hugo Pipes")

On the CSS side, `assets/css/main.css` holds ten `@import` statements, pulling in the
tokens, the two Chroma stylesheets, and the base, layout, component, content, block
and print layers in that order. `css.Build` inlines them into one file and rewrites
the `url()` references inside it; `resources.Concat` then appends the site's own
`assets/css/custom.css`. On the JavaScript side, `assets/js/main.js` imports seven
modules from `./modules/`, and `js.Build` hands that graph to esbuild, which emits one
file with `format = 'iife'` and `target = 'es2020'`. Either way the output is one file
fetched in one request.

## The CSS pipeline in full

`layouts/_partials/head/css.html` is the only place in the theme that builds a
stylesheet:

```go-html-template
{{- $parts := slice -}}
{{- with resources.Get "css/main.css" -}}
  {{- $parts = $parts | append (. | css.Build (dict "minify" false)) -}}
{{- else -}}
  {{- errorf "theme asset assets/css/main.css is missing — cannot build the stylesheet" -}}
{{- end -}}
{{- with resources.Get "css/custom.css" -}}
  {{- $parts = $parts | append (. | css.Build (dict "minify" false)) -}}
{{- end -}}

{{- $css := $parts | resources.Concat "css/bundle.css" -}}
{{- if hugo.IsDevelopment -}}
  <link rel="stylesheet" href="{{ $css.RelPermalink }}">
{{- else -}}
  {{- with $css | minify | fingerprint "sha384" -}}
    <link rel="stylesheet" href="{{ .RelPermalink }}" integrity="{{ .Data.Integrity }}" crossorigin="anonymous">
  {{- end -}}
{{- end -}}
```

Four details deserve naming. `css.Build` gets `minify` set to `false`, because
minification is left to the single `minify` call further down. The path in
`resources.Concat "css/bundle.css"` is a **resource buffer name**, not a file on disk.
The `errorf` turns a missing entry file into a failed build instead of a page that
merely renders unstyled. And a development build emits `css/bundle.css` while a
production build emits the hashed `css/bundle.min.<hash>.css`.

## The JS pipeline and the `@params` virtual module

The important part is the `params` value handed to `js.Build`:

```go-html-template
{{- $params := dict
      "searchIndex" $searchIndex
      "i18n" (dict "copy" (T "copy") "searchNoResults" (T "searchNoResults")) -}}

{{- $opts := dict
      "minify"    (not $dev)
      "target"    "es2020"
      "format"    "iife"
      "sourceMap" (cond $dev "linked" "none")
      "params"    $params -}}
```

That value is not a file and not an environment variable: esbuild exposes it as the
module `@params`, which is why `import * as params from '@params';` in
`assets/js/main.js` hands you the dictionary above. It is the only path in the site
from Hugo configuration into browser code — no JSON endpoint, no `data-` attributes
smeared across the `<body>`.

## Which parts need Node, and which do not

Most of it does not, because Hugo already does the work: `css.Build` uses a Go CSS parser to resolve `@import` and rewrite `url()`, `js.Build` embeds esbuild to do the bundling, and `resources.Concat`, `minify` and `fingerprint` are template functions. The design system, the script bundle, minification, hashing and SRI therefore need no `package.json` at all.

Exactly one part does: the Tailwind v4 stage goes through the official `css.TailwindCSS` integration, which runs the CLI installed with npm at the site root. That is why `package.json` lists two packages and why CI has exactly one `npm ci` step. The trade is deliberate — generating Tailwind on demand means scanning the rendered output, and nothing but Tailwind's own compiler does that accurately.

The cost is still clear: no SCSS, no PostCSS plugins, no arbitrary JavaScript transformers. What you get in exchange is that, apart from Tailwind, the build environment is a single binary.

## File-level merging: a same-named project asset replaces the theme's

This is the trap that bites hardest at theme-upgrade time. `assets/`, `layouts/` and
`static/` all merge at the **file level**: when two layers hold the same path, the
later layer replaces the earlier file outright. Ship an `assets/css/main.css` in your
project and it **replaces** the theme's file whole — later style fixes in the theme
then fail to reach the page, and the build reports nothing.

The fix is a different filename and composition by order rather than replacement.
That is what `assets/css/custom.css` does here: three overrides (for instance
`:root { --radius-lg: 16px; }`) concatenated after the theme output by `css.html`, so
they win by the cascade.

## Fingerprinting, SRI, and why the two files stay separate

The last step of a production build is `fingerprint "sha384"`, which writes the content
hash into the filename and produces the `integrity` value. A URL like
`css/bundle.min.573453…css` maps to exactly one body of content and can therefore be
cached as long as you like.

The `integrity` attribute is not decoration: the browser checks the body against the
hash and refuses to execute it when they disagree. Keeping the two files separate
instead of combining them is deliberate: editing one line of CSS does not invalidate
the cached JavaScript, or the other way round.
