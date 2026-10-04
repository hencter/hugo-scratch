+++
title = 'The CSS pipeline'
linkTitle = 'CSS'
description = 'How two compiled parts become one stylesheet: css.Build inlines the theme design system, Tailwind v4 emits the utilities actually used, and the light/dark switch reaches the Chroma code colours.'
date = 2026-02-25
weight = 10
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/', '/docs/assets/']
outcomes = ['Read the two compiled parts and the fixed concatenation order', 'Know which file your own styles belong in', 'Regenerate both Chroma stylesheets']
tags = ['CSS']
+++

`layouts/_partials/head/css.html` joins two unrelated compilations into one stylesheet: the theme's design system goes through `css.Build`, Tailwind goes through `css.TailwindCSS`, and both are then minified, fingerprinted and given an SRI hash. The pipeline is under ninety lines of template, and it decides several things at once — how many requests the browser makes, whether your changes survive a theme update, and whether code blocks change colour when the reader switches between light and dark.

## Two compiled parts, one stylesheet

The theme's design system starts at `assets/css/design-system.css`, which contains almost no rules of its own — only ten `@import` statements:

{{< filetree >}}
themes/hugo-scratch-theme/assets/css/
  design-system.css   ← design-system entry; imports only
  tokens.css          ← colours, spacing and fonts as custom properties
  chroma-light.css    ← light syntax-highlighting rules (generated)
  chroma-dark.css     ← dark syntax-highlighting rules (generated)
  base.css            ← reset, element defaults, accessibility primitives
  layout.css          ← grid and the three-column layout
  components.css      ← buttons, cards, table of contents
  content.css         ← prose typography inside .prose
  blocks.css          ← the home-page blocks
  releases.css        ← styles for the changelog page
  print.css           ← @media print
  tailwind.css        ← the Tailwind v4 entry point (the second part)
{{< /filetree >}}

The first part is the design system: `resources.Get "css/design-system.css"` and the site's own `css/custom.css` are joined by `resources.Concat` into `css/design-system-bundle.css`, then handed to `css.Build`, which inlines every `@import`; the template wraps the result in `@layer components { … }`. The second part is Tailwind: `resources.Get "css/tailwind.css"` goes to `css.TailwindCSS`, where the Tailwind v4 CLI expands `tailwindcss/theme.css` and `tailwindcss/utilities.css` and emits only the utility classes that are actually used.

The two parts are concatenated at the end, in a fixed order:

```text
@layer theme, components, utilities;   ← its own resource, pinned first
        ↓
[output of tailwind.css: theme + utilities]  →  [@layer components { design system }]
        ↓
resources.Concat "css/bundle.css"
        ↓
(production) minify → fingerprint "sha384"
        ↓
<link rel="stylesheet" href="…/css/bundle.min.<hash>.css" integrity="sha384-…" crossorigin="anonymous">
```

The layer-order statement is prepended by the template as its own resource rather than written inside `tailwind.css`, because an `@layer` statement only fixes precedence when it precedes every layered block. Written in the entry file it lands mid-file, and after minification there is even less of a guarantee — and then the `components` layer sits below `utilities`, `class="mt-0"` loses to `.prose p { margin }`, and no rule you can read explains why.

## Why two parts at all

One entry file cannot hold both kinds of import. The Tailwind CLI resolves `@import` relative to **the directory Hugo runs in** (the project root), which is what lets it understand a bare specifier such as `tailwindcss/theme.css`. A relative path such as `tokens.css` is understood only by Hugo's own inliner, which resolves against the asset tree — and that is also why a site-provided `assets/css/custom.css` can be found at all. Splitting the entries hands each import kind to the resolver that understands it; mix them in one file and the build stops at `Can't resolve 'tokens.css' in '<project root>'`, which also tells you the base is the run directory rather than the stylesheet's own directory.

## custom.css: the line between site and theme

The theme's `assets/` and the site's `assets/` share one namespace at lookup time, but the merge is at **file level**: a path exists once, and the site's copy silently replaces the theme's. Site styling therefore lives entirely in `assets/css/custom.css`, which is concatenated right after `design-system.css` in the first part — so it belongs to the same `components` layer and wins through the cascade rather than through `!important`.

{{< warning title="Never ship your own assets/css/design-system.css" >}}
That replaces the theme's stylesheet wholesale: the build succeeds, the page renders, the styling is gone, and the next theme update takes every fix with it. `custom.css` demonstrates the right move — override a custom property instead of rewriting rules:

```css
:root {
  --radius-lg: 16px;
}
```

One variable changes the cards, the code blocks and the callouts together, because all of them read the same token.
{{< /warning >}}

{{% steps %}}
{{% step "Append to custom.css" %}}
Small changes go straight into `assets/css/custom.css`. The pipeline already reads it, so no template needs to change.
{{% /step %}}
{{% step "Split the file and import it" %}}
Create `assets/css/home.css` and add `@import "home.css";` at the top of `custom.css`. `css.Build` inlines imports for `custom.css` exactly as it does for the design system, so the new file joins the first part and the output is still a single file.
{{% /step %}}
{{% step "Confirm the output" %}}
After `hugo --ignoreCache`, check whether the filename hash under `public/css/` changed. If nothing changed, check that the file is not sitting in `static/css/`, a directory that is copied verbatim and bypasses the pipeline completely.
{{% /step %}}
{{% /steps %}}

## How the light/dark switch is wired

The colour switch lives on two attributes of `<html>`, and both are required:

- `data-theme="light" | "dark"` — drives the custom properties in `tokens.css`, where the entire palette is one swapped block;
- `class="light" | "dark"` — drives the Chroma stylesheets, which were generated with a mode selector and therefore scope every rule as `.light .chroma …` and `.dark .chroma …`.

Both values are written before the first paint by the inline script in `layouts/_partials/head/theme-init.html` (which is why there is no flash of the wrong theme), and kept up to date by `assets/js/modules/theme.js` when the reader clicks the toggle. The two share the localStorage key `hugo-scratch:theme`; change one and you must change the other, or the choice appears to revert on reload.

One precondition makes code blocks follow along: `noClasses = false` under `[markup.highlight]` in `config/_default/hugo.toml`. It makes Chroma emit class names instead of inline styles. Set it to `true` and the colours are frozen into the HTML, where neither mode selector can reach them.

## Regenerating the Chroma stylesheets

`chroma-light.css` and `chroma-dark.css` are not hand-written; their first line records the command that produced them:

```bash
hugo gen chromastyles --style=github --mode light --modeSelector \
  --classLight light --classDark dark > themes/hugo-scratch-theme/assets/css/chroma-light.css

hugo gen chromastyles --style=github-dark --mode dark --modeSelector \
  --classLight light --classDark dark > themes/hugo-scratch-theme/assets/css/chroma-dark.css
```

`--modeSelector` wraps every rule in a top-level class, and `--classLight` / `--classDark` name that class; the names must match the `class` attribute on `<html>`. To change the palette, regenerate with a different `--style` value — print a candidate to the terminal first rather than overwriting a file and regretting it.

## Prerequisites and troubleshooting

The Tailwind part is not implemented by Hugo itself; it runs the CLI installed with npm at the site root. Several things therefore have to be true first, and the configuration already arranges them:

- `npm ci` has been run at the site root (`package.json` lists `tailwindcss` and `@tailwindcss/cli`);
- `[build.buildStats] enable = true`, plus the module mount that exposes `hugo_stats.json` as `assets/notwatching/hugo_stats.json` — Tailwind only emits utilities that **really appeared in the rendered output**, and this file is its input; the matching `[build.cachebusters]` entry is what makes a change in the statistics rebuild the stylesheet;
- `[security.exec] allow` includes `tailwindcss`, otherwise Hugo refuses to run it;
- `layouts/_partials/head.html` calls `head/css.html` from inside a deferred template: the statistics file only exists once every page has been rendered, and an inline call would compile against the previous build's data and fail outright on a clean clone.

Troubleshoot in the same order as that dependency list: no colour at all on the page means checking whether `public/css/bundle.min.<hash>.css` was produced; only Tailwind utilities missing means checking that `npm ci` ran and `hugo_stats.json` exists; only your own new rule ignored means checking that it lives in `custom.css` or a file that imports from it; source and output both correct but the browser still showing the old styling means suspecting something caching the HTML, since the fingerprint guarantees a new filename whenever content changes.

## Reference

{{< docref href="functions/css/" title="CSS functions" >}}
{{< docref href="functions/css/tailwindcss/" title="Tailwind CSS" >}}
