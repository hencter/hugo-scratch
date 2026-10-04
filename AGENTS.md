# AGENTS.md — the working contract for this repository

> 中文速览：**先 `npm ci`，再 `hugo server`**。改动内容只碰 `content/`，改动版式只碰
> `themes/hugo-scratch-theme/`。交付前必须跑通下面那条严格构建，并去 `public/` 里读产物
> 确认页面真的渲染了。`public/`、`resources/`、`hugo_stats.json` 是**产物**，永远不要手改。
> 详细的坑与约定都在本文件里，不需要再查别处。

This file is the single source of truth for working on this repository. It is written for an
agent that has to make a change and prove it. If something here disagrees with another
document, this file wins — and please fix the other document.

---

## 1. What this repository is

A bilingual (Simplified Chinese at `/`, English at `/en/`) Hugo site that uses the broad
common Hugo feature set on purpose, plus a **separate theme repository** wired in as a git
submodule at `themes/hugo-scratch-theme`.

| | |
| --- | --- |
| Site repository | `https://github.com/hencter/hugo-scratch` |
| Theme repository | `https://github.com/hencter/hugo-scratch-theme` |
| Published site | `https://scratch.hugozh.cn/` |
| Hugo floor | 0.146.0 (declared by the theme's `[module.hugoVersion]`) |
| Verified with | Hugo 0.167.0, **standard and extended** |
| Bilingual | `page.md` is zh-cn, `page.en.md` is its English twin |

`baseURL` is `https://scratch.hugozh.cn/` — a custom domain, served from the **root** of that
host — so no generated URL carries a path prefix. The prefix only appears on a GitHub Pages
*project* site published without a custom domain (`https://<owner>.github.io/<repo>/`); whichever
form is in use, `baseURL` must match it exactly, because a mismatch shows up as 404s for the
stylesheet and the scripts rather than as a build error.

---

## 2. Run it

```bash
git clone --recurse-submodules https://github.com/hencter/hugo-scratch.git
cd hugo-scratch
npm ci          # the Tailwind v4 CLI lives at the site root; the build needs it
hugo server     # http://localhost:1313/
```

Already cloned without submodules? `git submodule update --init --recursive`.

The submodule is not optional, and the way it fails does not look like a theme problem.
Measured on this repository:

- **cloned without `--recurse-submodules`** — the directory exists but is EMPTY, so the theme's
  `hugo.toml` is missing and the build stops while loading the configuration:
  `ERROR failed to create config: unknown output format "md" for kind "taxonomy"`.
  The theme is where those output formats are declared, so a missing theme surfaces as a
  *config* error that names no page at all.
- **directory deleted entirely** — `ERROR failed to load modules: module
  "hugo-scratch-theme" not found in "<path>"`.

`npm ci` is not optional either. The Tailwind stage of the stylesheet runs a CLI installed by
npm, not something Hugo ships. Without it the build stops with a missing `tailwindcss`
executable, and again the message names no page.

That last failure has two shapes, and the second one is the one CI platforms hit:

| What the build environment has | The error |
| --- | --- |
| nothing called `tailwindcss` | `You need to install TailwindCSS CLI … binary with name "tailwindcss" not found in PATH` |
| a `tailwindcss` that is not the npm CLI (a build image that ships the standalone binary) | `TAILWINDCSS: failed to transform "/css/tailwind.css" (text/css): binary "tailwindcss" is not a Node.js script` |

Both mean the same thing: the install step did not run. Hugo ≥ 0.161 only accepts the CLI
installed through npm, so a build must be `npm ci` **then** `hugo` — never `hugo` alone. Every
platform therefore needs two steps, and `npm run build` is the second one.

---

## 3. The gate: one command, and only exit 0 counts

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

Run it from the repository root before claiming any change is done. Each flag blocks a class
of defect that is otherwise invisible:

| Flag | What it catches |
| --- | --- |
| `--panicOnWarning` | the first WARNING becomes a failure: deprecated config keys and template methods, render-hook warnings |
| `--printPathWarnings` | two pages writing to the same output path (one silently overwrites the other) |
| `--printUnusedTemplates` | a template nothing reaches — usually a shortcode or render hook written but never called |
| `--printI18nWarnings` | a translation key used by a template but missing from a language file |
| `--ignoreCache` | rules out a stale file cache when a result makes no sense |

The render hooks add two more failures the flags cannot express on their own: a site-relative
Markdown link that resolves to no page, and an image that exists in none of the page bundle,
`assets/` or `static/`.

**The build output is the evidence.** A page rendered only if the directory exists:

```
public/docs/start/quick-start/index.html      # the rendered page
public/docs/start/quick-start/index.md        # its Markdown twin
public/en/docs/start/quick-start/index.html   # the English twin
```

A silent console does not mean a page rendered. `hugo list all` prints the content inventory
when the page count looks wrong.

Before publishing, the documentation also recommends the site audit, which surfaces problems
that are silent by default. Two exclusions are deliberate: **the site documents the very
strings this searches for** (this audit is explained in the feature-matrix, navigation and
Markdown pages), and **`public/search.json` embeds page bodies by design**, so it inherits
whatever those pages say. An unscoped grep fails on its own documentation.

```bash
HUGO_MINIFY_TDEWOLFF_HTML_KEEPCOMMENTS=true HUGO_ENABLEMISSINGTRANSLATIONPLACEHOLDERS=true hugo --ignoreCache

scan() {
  local needle="$1" why="$2" hits
  hits=$(grep -rn --binary-files=text --fixed-strings "$needle" public/ \
    | grep -v '/docs/reference/feature-matrix/' \
    | grep -v '/docs/configuration/navigation/' \
    | grep -v '/docs/content/markdown/' \
    | grep -v '/search.json:' || true)
  if [ -n "$hits" ]; then echo "::error::$why"; echo "$hits" | head -20; exit 1; fi
}

scan "HAHAHUGOSHORTCODE" "leaked shortcode placeholder"   # safe to spell in full: content containing it cannot build
scan "MISSING_TRANSLATION" "missing-translation placeholder"
scan "raw HTML omitted"   "HTML discarded because unsafe was off"
```

`.github/workflows/ci.yml` runs exactly this, so the local audit and CI cannot disagree.

### Building in parallel

Two agents must not build into the same directories. Isolate both the destination and the
cache, and note that `--cacheDir` must be an absolute path:

```bash
hugo -d .verify-a --cacheDir "D:/Projects/.hugo-cache-a" --ignoreCache --logLevel warn
```

`.verify*` and `.hugo-cache*` are gitignored.

---

## 4. Where things live

```
hugo-scratch/
├── AGENTS.md                  ← you are here
├── README.md                  human-facing overview
├── package.json               tailwindcss + @tailwindcss/cli (the only npm deps)
├── config/
│   ├── _default/              hugo.toml, languages.toml, params.toml, menus.<lang>.toml
│   └── production/            environment overrides, merged when hugo.Environment == production
├── content/                   one tree, both languages (.en.md suffix)
├── data/changelog.toml        source data for the /changelog/ content adapter
├── assets/css/custom.css      the SITE's stylesheet overrides (theme stays untouched)
└── themes/hugo-scratch-theme/ the theme (git submodule, its own repository)
```

The theme owns the design system, the templates and the shortcodes:

```
themes/hugo-scratch-theme/
├── hugo.toml                  params, outputformats, mediatypes (the only keys a theme may set)
├── assets/css/                design-system.css, tailwind.css, tokens.css, …
├── assets/icons/              IconifyJSON collections: lucide.json, local.json
├── assets/js/                 main.js + modules/
├── i18n/                      en.toml, zh-cn.toml — identical key sets
├── static/                    favicon.svg + PNG fallbacks, webmanifest, logo (no .ico)
└── layouts/
    ├── baseof.html            the document contract
    ├── home|page|section|taxonomy|term|404.html
    ├── list.md, page.md       Markdown output formats
    ├── home.llms.txt, home.search.json, home.pages.json
    ├── rss.xml, sitemap.xml, robots.txt
    ├── _partials/             head/, icons/, layout/, and the components
    ├── _shortcodes/           one file per shortcode
    └── _markup/               render-*.html, one per element kind
```

**Never edit**: `public/`, `resources/`, `.hugo_build.lock`, `hugo_stats.json`. They are
output or state; editing them is overwritten or makes the next build self-contradictory.

---

## 5. Use Hugo's own commands instead of guessing

Do not state a key, flag or default from memory. Ask the binary:

```bash
hugo version      # which behaviour applies
hugo config       # effective settings, defaults included
hugo list all     # the content inventory
hugo gen doc --dir .hugo-doc          # CLI reference for THIS version
hugo gen chromastyles --help          # regenerate the code-colour stylesheets
hugo env          # build/runtime information
```

`hugo config` in particular settles every argument about whether a key exists, what its
default is, and whether a theme-provided value survived. It is also where the valid
`[caches]` names appear — the older `getjson` is gone and aborts the build with
`"getjson" is not a valid cache name`.

---

## 6. Content contract

### Front matter — documentation page

```toml
+++
title = 'Full title'
linkTitle = 'Short title for the sidebar'
description = 'One sentence: what problem does this page solve?'
date = 2026-02-14
weight = 20
difficulty = 'beginner'        # beginner | intermediate | advanced
estimatedTime = 12             # minutes
prerequisites = ['/docs/start/']
outcomes = ['…', '…']
tags = ['Hugo']
+++
```

Facts live in the front matter, not in prose, because four consumers read them: the on-page
facts panel, the sidebar, `pages.json`, and the `.md` twin. An agent can decide whether to
follow a page or look a fact up without fetching the HTML.

### Rules

- Body starts at `##`. The template renders the `<h1>`; a second level-one heading gives the
  page two competing top-level headings.
- `weight` must be unique inside its section, spaced by 10. Section weights: `docs` 10,
  `start` 10, `configuration` 20, `content` 30, `templates` 40, `assets` 50, `seo` 60,
  `deploy` 70, `agents` 80, `reference` 90, `blog` 20, `changelog` 30, `legal` 90.
- Internal links are root-relative (`/docs/start/`) and must resolve to a real page. A link that
  does not resolve fails the strict build. `prerequisites` entries are page paths too.
- Every page has a twin. The English file is the same directory and base name plus `.en`:
  `quick-start.md` ↔ `quick-start.en.md`, `_index.md` ↔ `_index.en.md`,
  `index.md` ↔ `index.en.md`. Keep heading structure, shortcode calls and code blocks in step.
- Per-language menus are config, not i18n: `config/_default/menus.zh-cn.toml` and
  `menus.en.toml`. Menu labels are deliberately NOT passed through `T`, because a missing key
  would warn on every page.

---

## 7. The iron rules of authoring

A violation fails the whole build, not one page.

1. **Never leave an unescaped shortcode delimiter in prose or inside a code fence.** To *show*
   shortcode syntax, escape it:
   `{{</* note */>}}` … `{{</* /note */>}}`, `{{%/* tabs */%}}` … `{{%/* /tabs */%}}`,
   closing forms `{{</* /name */>}}` and `{{%/* /name */%}}`.
   To show the escape itself: `{{</*/* note */*/>}}`.
2. **Never write Hugo's shortcode placeholder prefix in full inside `content/`.** The literal
   string aborts rendering with `illegal state in content; shortcode token missing end delim`,
   attributed to whichever page happens to be rendering — not necessarily the page containing
   it. To display it, break it with a zero-width entity outside a code span
   (`H&#xfeff;AHAHUGOSHORTCODE`). This does not apply to `AGENTS.md`, README or the workflows,
   which are not content — and grepping the BUILD OUTPUT for the full literal is safe and
   correct, because a site that contains it cannot build at all.
3. **`{{% %}}` shortcodes need `unsafe = true`.** Their output is re-parsed as Markdown. With
   `[markup.goldmark.renderer] unsafe` off, every panel is replaced by
   `<!-- raw HTML omitted -->`.
4. **Be careful with shortcode syntax inside a standard-notation shortcode body.** The theme
   pipes such a body through `markdownify`, and that second render pass treats your example as
   a real call. Which failure you get depends on the example: a SELF-CLOSING one such as
   `{{</* badge "x" */>}}` quietly renders as a badge (probably not what you meant, but the
   build passes), while one that needs a closing tag such as `{{</* tip "…" */>}}` aborts with
   `failed to extract shortcode: shortcode "x" must be closed or self-closed`. Either way, put
   examples in ordinary prose or in a code fence outside the callout.
5. **Never mix a positional and a named parameter in one shortcode call.** `{{< badge "Beta"
   tone="accent" >}}` fails with `cannot mix named and positional parameters`. All positional
   or all named.
6. **Do not put a glob containing `*/` inside a Go template comment.** `public/*/index.html`
   ends the comment early and the template fails to parse with
   `comment ends before closing delimiter`.
7. **`{{ with .Date }}` is always true.** `.Date` is a struct, so an undated page happily
   prints `0001-01-01`. Guard with `.IsZero`. Print machine-readable dates as
   `<time datetime="…">` holding ISO 8601, whatever the visible text says.

---

## 8. Traps this repository actually hit

Each of these cost a real debugging cycle. They are recorded because the error message points
somewhere other than the cause.

| Symptom | Real cause |
| --- | --- |
| `found no layout file for "html" for kind "page"` | the theme submodule is not checked out |
| `frontmatter.theme` silently ignored, no theme applied | a bare TOML key written **below** a `[table]` header — it joins that table. Scalars first, tables after |
| `"getjson" is not a valid cache name` | the cache was renamed; `hugo config` lists the current names |
| `illegal state in content; shortcode token missing end delim` | the shortcode placeholder prefix appears literally in content |
| `comment ends before closing delimiter` | a `*/` sequence inside a Go template comment |
| `can't evaluate field TFoot in type tables.tableContext` | `.TFoot` does not exist on the 0.167 table render-hook context; a Markdown table has no footer |
| `index of type string with args [map[…]]` | `dict … \| index $type` reverses the arguments; write `index (dict …) $type` |
| `cannot mix named and positional parameters` | a shortcode call mixing `"value"` with `key="value"` |
| `shortcode "x" must be closed or self-closed` | an escaped example inside a standard-notation body that is `markdownify`-ed, when the example names a shortcode that needs a closing tag. A self-closing example renders instead of failing |
| `Can't resolve 'tokens.css' in '<project root>'` | Tailwind resolves imports and `@source` relative to the **directory Hugo runs in**, not the stylesheet. Relative imports belong to `css.Build`; bare specifiers belong to Tailwind |
| Tailwind utilities never generated | measured: for the classes THIS site uses, deleting `@source "hugo_stats.json"` produces a byte-identical bundle, because every class also occurs literally in a file Tailwind auto-scans. Keep the line regardless — it is what covers a class that exists only in the rendered output, assembled by interpolation, which auto-scanning cannot see |
| a utility loses to a component class | the `@layer` order statement must be the FIRST thing in the bundle; a mid-file statement is rewritten by the minifier |
| `--cacheDir` must be absolute | it cannot be a relative path |
| content adapter produced `/changelog/v1-0-0.en/` | content adapters create pages in the **default language only**; a language suffix in `path` becomes part of the URL. The English section renders the same data through the `{{< changelog >}}` shortcode instead |
| a `.Date.Format` call fails on data | TOML dates arrive through `hugo.Data` as an untyped value; normalise with `time.AsTime` |
| a build carries `noindex, nofollow` | the environment was not production. Plain `hugo` **is** production and emits `index, follow, …`; only `hugo server` and `hugo -e development` are development, and `layouts/robots.txt` disallows every crawler for the same reason |
| every code block renders monochrome, all its lines run together, and `linenos` silently does nothing | the code-block render hook printed `transform.HighlightCodeBlock .`.Inner. `.Inner` is the highlighted code and nothing else — no `<pre>`, no `<code>`, no `.chroma` element — so the newlines collapse and no `chroma-*.css` selector matches. Print `.Wrapped`, which is the complete block |
| no page ever gets a table of contents although `showTableOfContents = true` and the page clearly has enough headings | `.Fragments.Headings` is a **one-element** slice wrapping a synthetic level-0 root (measured on 0.167, and reproduced in a theme-less throwaway site); the real headings are that node's `.Headings`. Count or walk the root's children, not the slice |
| `[ui] showTableOfContents`, `tocMinHeadings` or `showSidebar` appear to be ignored | `index $params "camelCase"` is case-SENSITIVE, while Hugo stores param keys lower-cased — the lookup returns nil and the `default` quietly hides it. Use field access (`$ui.tocMinHeadings`), which is case-insensitive, and change the value once to prove the page count moves |
| a `[^1]` footnote marker shows up **literally** in the rendered page, with no footnote block | the marker sits inside a `{{% %}}` shortcode body. That body is rendered in its own pass, so the definitions elsewhere on the page never pair with it. Footnotes belong in ordinary prose; inside a `{{% %}}` body keep the link inline |
| the browser tab shows Hugo's default placeholder icon although the theme ships a brand mark | nothing in `<head>` declared an icon, so the browser fell back to its own request for `/favicon.ico` — and that file was byte-identical to the placeholder `hugo new theme` scaffolds (measured: identical SHA256, 15406 bytes). Declare `favicon.svg` and the PNG fallbacks in `_partials/head/icons.html` |
| `Resize`/`Fill` fails on an SVG, and no Hugo command can write an `.ico` | Hugo does not rasterise SVG (`resource "/x.svg" of media type "image/svg+xml" does not support this method`), and `hugo gen` offers only chromastyles/doc/man. Raster icons must be committed assets or produced outside the build — which is why the icon data is vendored JSON read by a partial, not fetched or generated at build time |
| `binary "tailwindcss" is not a Node.js script` | the build ran `hugo` with no install step, so Hugo picked up whatever `tailwindcss` the build image ships (the standalone binary) — Hugo ≥ 0.161 accepts only the npm CLI. Install first; see §2 |

Deprecations that fail under `--panicOnWarning`, with their replacements:

| Deprecated | Use |
| --- | --- |
| `.Page.IsNode` | `.IsPage` / `.IsBranch` |
| `.Site.Data` | `hugo.Data` |
| `.Site.Sites` / `.Page.Sites` | `hugo.Sites` |
| `.Language.LanguageName` | `.Language.Label` |
| `languageCode` / `languageName` | `locale` / `label` |
| `cascade._target` | `cascade.target` |
| `build._build` | `build` with `list` / `render` / `publishResources` |
| `imaging.quality` | `imaging.jpeg.quality`, `imaging.webp.quality`, … |

---

## 9. Common tasks

**Add a documentation page.** Create the pair, then build and read `public/`.

```bash
hugo new content docs/configuration/foo.md       # uses the theme's docs archetype
# write content/docs/configuration/foo.en.md with the same shape
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

Give it a unique `weight` in its section. If it should appear in the sidebar of another
section, add that section to `[params.nav] sidebarSections`.

**Add a shortcode.** `themes/hugo-scratch-theme/layouts/_shortcodes/<name>.html`. Decide the
notation first: `{{< >}}` runs after the Markdown renderer (`.Inner` is raw text — call
`markdownify`), `{{% %}}` runs before it (`.Inner` is rendered HTML, and its headings reach the
table of contents). Then **call it from exactly one page**, or `--printUnusedTemplates` fails
the build.

**Add a render hook.** `themes/hugo-scratch-theme/layouts/_markup/render-<element>.html`. The
directory is `_markup`; a file in the wrong place is never called and never reported.

**Add an icon.** The paths live in IconifyJSON collections under
`themes/hugo-scratch-theme/assets/icons/`, and `_partials/icon.html` reads them with
`resources.Get` + `transform.Unmarshal` (`_partials/icons/set.html`, called through
`partialCached`, parses each collection once per build). Call it with a bare name from the
default set or with `prefix:name`; `local:logo` is the brand mark, which belongs to no set.
To add one, copy its body from `https://api.iconify.design/<set>.json?icons=<name>` into the
collection — there is no script, no npm package and no network access in the build, and a name
that is missing warns with exactly that URL.

**Override something from the site.** Put a file of the same path under the site's `layouts/`,
`assets/` or `static/`. Theme and project merge at FILE level, so a same-named file *replaces*
the theme's — which is why the site's stylesheet is `assets/css/custom.css` and not
`design-system.css`.

**Add a machine-readable output.** Declare the format in the theme's `hugo.toml` (only
`params`, `menu`, `outputformats`, `mediatypes` are honoured there), switch it on in the site's
`[outputs]`, and add the template named `<kind>.<format>.<suffix>` — for example
`home.search.json`. Set `notAlternative = true` to keep it out of `rel="alternate"`.

**Change the design.** Edit the theme's `assets/css/*.css`, not `custom.css`, unless the change
is site-specific. Regenerate the code colours with:

```bash
hugo gen chromastyles --style=github      --mode light --modeSelector --classLight light --classDark dark > assets/css/chroma-light.css
hugo gen chromastyles --style=github-dark --mode dark  --modeSelector --classLight light --classDark dark > assets/css/chroma-dark.css
```

---

## 10. Publishing

`.github/workflows/` holds two workflows. Both must run `npm ci` before the build — the
Tailwind CLI is not part of Hugo:

```bash
actions/setup-node + npm ci
npm run build      # = the strict gate in §3
actions/upload-pages-artifact + actions/deploy-pages
```

Any other build platform needs the same two steps in the same order: an install step that runs
`npm ci`, then a build step that runs `npm run build`. The script exists so the strict command
lives in one place and so the platform puts `node_modules/.bin` on PATH. A platform whose build
command is a bare `hugo`, with no install step, fails with the `tailwindcss` errors in §2 — the
build image's own standalone `tailwindcss` is not acceptable to Hugo ≥ 0.161.

The checkout needs `submodules: recursive` (for the theme) and `fetch-depth: 0` (because
`enableGitInfo = true` makes "last updated" a fact from the commit history). Repository
settings: Pages → Source = GitHub Actions.

---

## 11. Definition of done

- [ ] The strict build in §3 exits 0, with no warning you cannot explain.
- [ ] Every page you touched has a rendered counterpart under `public/`, in both languages.
- [ ] Its `.md` twin exists and opens with valid YAML.
- [ ] No inbound link still points at an old path (the render hook would say so).
- [ ] `hugo_stats.json` changed if you changed class names, and the CSS bundle was regenerated.
- [ ] Any claim about a default, a flag or a version was verified against this Hugo, not recalled.
- [ ] If two agents worked in parallel, the Lead ran the strict build again on the merged tree.
