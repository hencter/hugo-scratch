+++
title = 'The JavaScript pipeline'
linkTitle = 'JavaScript'
description = 'How js.Build turns main.js into a single IIFE, how @params injects the search-index URL and the UI strings, and what each of the seven modules owns.'
date = 2026-02-25
weight = 20
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/', '/docs/assets/']
outcomes = ['Explain what @params puts in the browser', 'Know what each of the seven modules owns', 'Add a module and wire it into the entry point']
tags = ['JavaScript']
+++

This pipeline looks more involved than the CSS one, but it really does one extra thing: it bundles the site's configuration and translations into the script. Once that clicks, `@params`, the two language-specific bundles, and the fact that the search box needs no data attributes at all turn out to be the same answer.

## One entry point, bundled by esbuild

`layouts/_partials/head/js.html` reads exactly one file — `assets/js/main.js` — and hands it to `js.Build`. esbuild runs inside Hugo, collapses the whole `import` graph into a single file, and the browser therefore makes one script request with no module loader involved. The options differ between development and production:

{{% tabs %}}
{{% tab "hugo server (development)" %}}
```text
minify=false   target=es2020   format=iife
sourceMap="linked"   params={…}
```
The output keeps a linked source map, so a console error points back at the original module instead of a line in the bundle. The script is loaded as a plain `<script src="…" defer>` with no fingerprint.
{{% /tab %}}
{{% tab "hugo (production)" %}}
```text
minify=true    target=es2020   format=iife
sourceMap="none"     params={…}
```
The bundle is minified and then run through `fingerprint "sha384"`; `integrity` and `crossorigin="anonymous"` land on the `<script>` tag. `defer` is present in both cases, so the script never blocks parsing.
{{% /tab %}}
{{% /tabs %}}

The target is `es2020` and the format is `iife`: the output is one immediately invoked function that attaches nothing to the global scope and cannot collide with another script on the page. To confirm the bundle exists, look for `public/js/main.<hash>.js`.

## @params: configuration and translations inside the code

Hugo turns `js.Build`'s `params` option into a virtual module named `@params`, and the entry point pulls the whole thing in on its first line:

```js
import * as params from '@params';
```

Only two things go into that dictionary. The first is the search-index URL, taken from the home page's `search` output format `RelPermalink` and wrapped in `with` — if a site removes `search` from `[outputs]`, this quietly becomes an empty string instead of failing the build. The second is an `i18n` sub-dictionary: six UI strings (copy, copied, search loading, no results, result count, search hint) are resolved at build time with `T` and written straight into the code.

{{< note >}}
`params` is injected at **build time**; it is not a runtime configuration channel. After editing a translation in `i18n/` or a value in `params.toml` you must rebuild for the script to carry the new value. The flip side is that, because the strings are frozen when the bundle is produced, the browser never spends a second request fetching a few UI labels.
{{< /note >}}

## What each of the seven modules owns

`assets/js/main.js` does three things only: import the modules, import `@params`, and call each module's `init` once the DOM is ready. The real work lives in `assets/js/modules/`:

`theme.js`
: The three-state switch — light, dark, follow the system — writing both `data-theme` and the `light`/`dark` class. It does not own the initial value: the inline script in `head/theme-init.html` has to run before the first paint, and a bundled module is deferred by definition.

`nav.js`
: Toggles the visibility of the mobile navigation and nothing else. The header's `<nav>` and menu list are real server-rendered markup, so with JavaScript disabled the navigation is still usable and still crawlable.

`search.js`
: Fetches `search.json` the first time a reader opens the search dialog. Matching is case-folded substring comparison, because Chinese, Japanese and Korean text has no word boundaries for a tokeniser to use.

`toc.js`
: The scroll highlight for the table of contents, built on `IntersectionObserver` rather than a `scroll` listener, so reading costs the main thread nothing extra.

`copy.js`
: The copy button on code blocks. The button itself is emitted server-side by the code-block render hook, so when the bundle fails to load it simply stays inert instead of disappearing.

`tabs.js`
: Adds keyboard navigation to tab sets and hides the unselected panels. Every panel is already in the HTML, so a crawler sees the complete content.

`back-to-top.js`
: Reveals the back-to-top button after roughly one viewport of scrolling by setting one data attribute; all styling stays in CSS.

## Why each language gets its own bundle

`params` carries translated strings, and each language has different ones, so the result of `js.Build` differs too: a bilingual build leaves two files with different hashes in `public/js/`, one referenced by the Chinese pages and one by the English pages.

That is not waste. Move the strings out of the script and you must either hang `data-label-*` attributes on every button in the markup or make the page fetch a language pack — the first spreads copy through the HTML, the second adds a round trip to the critical path. Build-time injection pays the cost once, at build time, and hands the reader a self-contained file. Fingerprinting lets the two outputs cache independently: editing a Chinese string changes only the Chinese filename.

## Adding a module

The procedure is fixed and touches two places:

{{% steps %}}
{{% step "Create the module file" %}}
Add a file under `assets/js/modules/` and export one initialiser, for example `export function initFoo() { … }`. Modules other than the entry point should not attach global listeners or assume the DOM is ready — that is the entry point's job.
{{% /step %}}
{{% step "Wire it into the entry point" %}}
In `assets/js/main.js` add `import { initFoo } from './modules/foo.js';` and call `initFoo()` inside the `ready(() => { … })` callback. `ready` already handles the case where the script loads after the DOM has been parsed.
{{% /step %}}
{{% step "Rebuild and confirm" %}}
Run `hugo --ignoreCache` and look at `public/js/`: a changed filename hash means the new module made it into the bundle. If the script complains that an element is missing, first confirm the data attribute it hooks onto is actually being rendered by a template.
{{% /step %}}
{{% /steps %}}

## What this pipeline deliberately avoids

Two things are worth stating plainly, because they are easy to import from other sites by habit. First, this part of the pipeline is not deferred: in `layouts/_partials/head.html` only the stylesheet sits inside `templates.Defer` (Tailwind has to wait for `hugo_stats.json` to be complete), while `head/js.html` is called directly — it emits its tag while `<head>` is parsed, and `defer` only decides when the script executes. Second, there is no front-end framework and no hydration — each module finds nodes, attaches a listener and sets an attribute, and the page is a readable, clickable document before the script loads. If you want an interactive component with a state tree, decide first whether it is worth breaking that premise.[^1]

[^1]: Upstream documentation: [js.Build](https://gohugo.io/functions/js/build/)
