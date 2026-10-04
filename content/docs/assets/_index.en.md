+++
title = 'The asset pipeline'
linkTitle = 'Assets'
description = 'How CSS and JavaScript travel from assets/ to a single file in the browser: Hugo Pipes builds, content fingerprints, SRI, and the boundary between theme and site assets.'
weight = 50
+++

This chapter answers two questions: where the stylesheets come from, and where the scripts come from. Both take the same shape of pipeline, but with different options at every step and completely different symptoms when they break — lose the stylesheet and the whole page collapses; break the script and usually one button stops responding, which is easy to blame on the browser.

## The two pages in this section

[The CSS pipeline](/docs/assets/css/) starts at `themes/hugo-scratch-theme/assets/css/design-system.css` and explains how `css.Build` inlines ten `@import` statements into the design system, how the Tailwind v4 part emits only the utilities that are really used, where the site's `assets/css/custom.css` is concatenated, why the light and dark Chroma stylesheets have to come out of `hugo gen chromastyles`, and which file a site's own rules actually belong in.

[The JavaScript pipeline](/docs/assets/js/) takes the other path: `assets/js/main.js` is bundled by `js.Build` into a single IIFE, the `@params` virtual module injects the search-index URL and the translated UI strings straight into the code, and seven modules each own exactly one job. After reading it you can decide which module a new interaction belongs in instead of piling more onto `main.js`.

## Who compiles this pipeline

There is no PostCSS configuration file and no Sass here: the `@import` statements of the design system are inlined by `css.Build`, and the scripts are bundled by `js.Build` — both built into Hugo, and both able to run on a non-extended binary. `[module.hugoVersion]` in `themes/hugo-scratch-theme/hugo.toml` says `extended = false` with `min = '0.146.0'`.

The one part that needs an external tool is Tailwind: `css.TailwindCSS` runs the Tailwind v4 CLI installed with npm at the site root, so `npm ci` has to run before a build, and `[build.buildStats] enable = true` plus the mount that exposes `hugo_stats.json` to the asset pipeline have to be in place — Tailwind only emits utilities that really appeared in the rendered output, and that file is its input. All of it lives in `config/_default/hugo.toml`, and the comment in `head/css.html` repeats it because none of the pieces is optional.

## How theme and site assets merge

`assets/` is Hugo's resource directory. The theme's `assets/` and the site's `assets/` behave as one namespace at lookup time, but the merge happens at **file level**: a given path exists once, the site's copy wins over the theme's, and the replacement is silent.

{{< note >}}
That is exactly why site styling lives in `assets/css/custom.css`. If the site shipped an `assets/css/design-system.css`, it would replace the theme's stylesheet wholesale: the build reports nothing, but the page looks as though the theme is broken. Scripts behave the same way: to add a module, create a site file next to the theme's `assets/js/modules/` and import it from the entry point; dropping in an `assets/js/main.js` swaps out the theme's entry point entirely.
{{< /note >}}

## What the build produces

After a build, only three kinds of artifact in `public/` belong to this section, and their names come from the `resources.Concat` name plus the `fingerprint` hash:

```text
public/css/bundle.min.<hash>.css   ← design system + Tailwind, one file, with SRI
public/js/main.<hash-a>.js         ← the Chinese script (@params carries zh strings)
public/js/main.<hash-b>.js         ← the English script (@params carries en strings)
```

There is one CSS file because the two compiled parts are concatenated at the end; there are two scripts because the UI strings travel inside the code, so each language compiles once. Together those two facts explain something practical: editing one style rule hands every page the same new hash, while editing one UI string renames only one language's script and leaves the other language's cache intact.

To confirm a class name made it into the output, search `public/css/bundle.min.<hash>.css` for it. Not finding it usually means the class never appeared in any rendered page rather than that the pipeline dropped it — Tailwind only emits utilities it scanned, which is exactly why `hugo_stats.json` has to take part in the build.

## How to verify what this chapter claims

After changing styles or scripts, make sure the dependencies are installed and then run a one-shot build:

```bash
npm ci
hugo --ignoreCache
ls public/css public/js
```

You should see one `public/css/bundle.min.<hash>.css`, and two `public/js/main.<hash>.js` — the UI strings are bundled into the script, so each language gets its own artifact. The hash in the filename is the content fingerprint — change the content and the filename changes, so browser caches never need a hand-written `?v=` to invalidate. If the CSS file is missing, one of the two compiled parts failed, and the build log will say which one.

Keep the two failure classes apart. A build-time failure always means something is missing: `head/css.html` names the resource it could not find, and the Tailwind part stops early when `npm ci` has not run or `hugo_stats.json` does not exist. A runtime "my styles did not apply" is usually unrelated to the artifact — the fingerprint guarantees a new filename when content changes, but it cannot stop you from opening a cached copy of the previous HTML.

Scripts behave the same way: a missing artifact is named by the `errorf` in `head/js.html`, while "a button does not respond" usually means the template never rendered the element the module hooks onto. Confirm that the data attribute really appears in the HTML under `public/` before reading the module code — the other order wastes a lot of time.
