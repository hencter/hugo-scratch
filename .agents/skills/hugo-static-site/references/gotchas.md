# Hugo trap catalogue

Each entry: what you see → what actually happened → what to do. Entries are referenced from
`SKILL.md` as G1, G2, …

## How to read this catalogue

- **documented** — reproduces stated behaviour and names the page or command involved: G1,
  G3–G10, G14, G18–G20.
- **observed** — reproduced on a working site, not stated in the docs: G11–G13, G15–G17, G21–G26.
- **not documented** — real but unstated; G2 says so in its own Status line.

Where an entry disagrees with what your own build shows, trust the build: these are behaviours,
not promises.

## G1 — `failed to extract shortcode: template for shortcode "x" not found`

**Symptom.** Build aborts during content assembly; the message names a shortcode nobody wrote.

**Cause.** Content contains a bare `{{<` or `{{%`. Hugo scans for shortcodes **before** Markdown,
and the documentation never exempts code blocks: its escaping rule is demonstrated *inside* a
fenced example, and every shortcode call in the docs is escaped
(<https://gohugo.io/content-management/syntax-highlighting/#escaping>,
<https://gohugo.io/contribute/documentation/#escaping-shortcode-syntax>). Never rely on a fence or
an inline code span to protect the syntax. The page that reports the error is usually the first
offender by line order, not the only one.

**Fix.** Rewrite each occurrence as an escape (`{{</* name */>}}`, `{{%/* name */%}}`,
`{{</* /name */>}}`), then re-scan the whole content tree rather than just the reported page.

## G2 — `illegal state in content; shortcode token missing end delim`

**Symptom.** A page that contains **no braces at all** fails to render; the error is attributed to
that page; rewriting the page does not help; moving the page out of `content/` makes the build pass.

**Cause.** Content contains the literal string `HAHAHUGOSHORTCODE`. That is the prefix Hugo uses
for its internal shortcode placeholders; content containing it drives the lexer into an illegal
state. The upstream Hugo documentation dodges this by writing `H&#xfeff;AHAHUGOSHORTCODE` — a
zero-width U+FEFF splitting the literal.

**Fix.** Replace the literal with `H&#xfeff;AHAHUGOSHORTCODE`, or restructure the sentence so the
full string never appears. The `&#xfeff;` entity must sit **outside** a code span: inside
backticks it is not decoded and the reader sees the entity.

**Status.** This one is **not documented** anywhere except the workaround in the official audit
page (<https://gohugo.io/troubleshooting/audit/>); it is a Hugo implementation detail. It is also
the only failure in this catalogue that Hugo attributes to the wrong cause, so it is worth knowing
even though the mechanism is inferred rather than specified.

**Why it hides.** The string looks like ordinary prose, and `grep` for `{{` finds nothing.

## G3 — deprecation warning: `languageCode` → `locale`

**Symptom.** `WARN deprecated: project config key languageCode was deprecated in Hugo v0.158.0 …
Use locale instead.`

**Fix.** Rename the key (`locale = "zh-cn"`). Keep `defaultContentLanguage` — it is a different
key and still current. If the site must also build on pre-0.158 Hugo, pick one key and document
the version requirement instead of shipping both.

## G4 — `hugo new site` no longer scaffolds content

**Symptom.** Scripts copied from older tutorials produce an unexpected skeleton.

**Cause.** Current Hugo scaffolds with `hugo new project <path>`; `hugo new site` is legacy.
Likewise `hugo` (build) is documented as `hugo build`, with `hugo` kept as the alias.
Check `hugo <cmd> --help` for the installed version instead of trusting a tutorial.

## G5 — front matter `_target` and `_build`

- `cascade` uses `target` (page matcher) with the alias `_target` deprecated; `lang` inside the
  matcher was deprecated in 0.153.0 in favour of the `sites` matrix.
- Page build options use `build`; `_build` is the old spelling. `list` = `always|local|never`,
  `render` = `always|link|never` (strings), `publishResources` = bool.

## G6 — render hook templates not found

**Symptom.** Hook files exist but Hugo ignores them.

**Cause.** Directory moved. Current: `layouts/_markup/render-*.html` (plus `layouts/_partials/`,
`layouts/_shortcodes/`). Legacy: `layouts/_default/_markup/`. `_default` is no longer used by the
current template system.

## G7 — image resource methods

`.Exif` was deprecated in 0.155.0 in favour of `.Meta` (0.155.3), and converting an image does
not carry metadata forward. `[imaging.exif]` keys (`disableDate`, `disableLatLong`,
`excludeFields`, `includeFields`) still configure what gets read.

## G8 — syntax-highlighting keys

- Fence options: `lineNos`, `lineNoStart`, `hl_lines`, `style` — not `linenos`/`linenostart`.
- `[markup.highlight]`: key names and defaults matter (`noClasses`, `style`, `wrapperClass`,
  `lineNumbersInTable`, `anchorLineNos`, `guessSyntax`, `codeFences`, `tabWidth`). Verify against
  the installed version; several defaults changed. `hugo gen chromastyles` emits a stylesheet
  when you switch to class-based highlighting.

## G9 — menu templates: `IsMenuCurrent` takes a Menu object

`PAGE.IsMenuCurrent MENU MENUENTRY` — the first argument is the entry's `.Menu` object, **not**
the menu name string (that was the old API). Entries must come from front matter or a config
entry with `pageRef` for menu/page association to work.

## G10 — Go template `and`/`or` do not short-circuit

`{{ if and $p $p.Title }}` still evaluates `$p.Title` and panics when `$p` is nil, because
`and` is a function. Precompute with `with`, or compute a safe value first.

## G11 — unguarded resource pipeline kills the build

`resources.Get "css/x.css" | minify | fingerprint` panics once the file is missing or renamed.
Wrap it in `with` and fail with `errorf` so the message names the missing asset.

## G12 — `with .Date` is always true

`time.Time` is a struct, so truthiness never fails. Use `{{ if not .Date.IsZero }}` before
formatting, or a page without a date prints `0001-01-01`.

## G13 — the build log is from a half-written file

**Symptom.** A failure that disappears on the next build without any edit, or an error against a
file whose content cannot produce it.

**Cause.** A watcher (`hugo server`) rebuilt while files were still being written. Editing while
a watcher runs guarantees these ghosts.

**Fix.** Stop the watcher (or serialize the work), then rebuild with `--ignoreCache`. Treat any
log captured during writes as unverified.

## G14 — duplicate or missing section weights

Section order comes from `_index.md` weight. Two sections sharing a weight order unpredictably;
a moved page keeps its old weight and can collide with a sibling. Re-check weights after any
file move.

## G15 — pages that moved but links did not

Moving or renaming a page changes its URL; every inbound root-relative link is now dead. Hugo
does not warn. Grep for the old path (`/old/section/page/`) across content and rewrite all hits
in one pass, then re-run the checker.

## G16 — `public/` lies in both directions

- Stale files remain after a page is deleted or renamed; prune with `--cleanDestinationDir`.
- Conversely, a page missing from `public/` means it never rendered — the fastest way to prove a
  long-standing failure that the console only hints at.

## G17 — CJK headings and anchors

Hugo keeps CJK characters in ids and drops punctuation, so `## 草稿、将来与过期内容` yields
`#草稿将来与过期内容`. An anchor copied from a GitHub-style renderer (`#cascade-1`) will not
match. Confirm ids from the rendered HTML.

## G18 — empty taxonomy pages

Default `tags`/`categories` taxonomies produce `/tags/` and `/categories/` even with no terms.
Either accept them or `disableKinds = ["taxonomy", "term"]`.

## G19 — `baseURL` and subpath deployment

Root-relative links (`/a/b/`) assume the site is served from the domain root. Deploying under a
subpath requires `baseURL` to include it, otherwise every internal link 404s. `hugo server`
usually masks the problem because it serves from `/`.

## G20 — `resources.Get` vs page resources

`resources.Get` reads `assets/`; files inside a leaf bundle are page resources reachable via
`.Resources`. A file in `static/` is copied verbatim and cannot be processed by the asset
pipeline.

## G21 — a "top-level" key silently becomes part of the table above it

**Symptom.** `theme = [...]` looks right in `hugo.toml`, yet the build warns
`found no layout file for "html" for kind "…"` for every kind, `--printUnusedTemplates` reports
`/baseof.html is unused`, and the page count collapses (226 → 21 in the case that found this).

**Cause.** TOML scoping. After a `[table]` header, every bare key belongs to that table, so a key
written *below* a table is no longer top-level:

```toml
[frontmatter]
  lastmod = [':git', 'lastmod']
theme = ["a", "b"]        # ← this is frontmatter.theme, not theme
```

**Fix.** Keep every top-level scalar above the first `[table]` header and add new tables at the
bottom. When a config value seems ignored, dump the effective config with `hugo config` and find
where the key actually landed.

## G22 — localized `:date_*` tokens fall back to English for some locales

**Symptom.** `{{ .Date | time.Format ":date_long" }}` renders `October 1, 2026` on a site whose
`locale` is `zh-CN`. No warning; the only clue is the output.

**Cause.** Hugo resolves the locale from `locale` (falling back to the language key) and hands it to
`bep/golocales`. The localized token tables do not cover every locale, and unsupported ones fall
back to English.

**Fix.** Do not rely on the token for CJK output. Use an explicit Go layout, ideally supplied by a
locale overlay theme:

```toml
[params]
  dateFormat = "2006年1月2日"
```

```go-html-template
{{ $layout := site.Params.dateFormat | default ":date_long" }}
{{ time.Format $layout .Date }}
```

**Observed:** control test on one template — `locale = "de-DE"` produced `1. Oktober 2026` while
`locale = "zh-CN"` produced `October 1, 2026`, so the configuration was right and the data is
incomplete.

## G23 — the file looks byte-perfect and Hugo still rejects it

**Symptom.** A config file (or a front matter block) that reads correctly in an editor makes Hugo
fail at load with a character that is plainly not in the text:

```text
failed to load config: "…/hugo.toml:1:1": unmarshal failed:
toml: invalid character at start of key: U+00FF 'ÿ'
```

**Cause.** Encoding, not syntax. `echo "…" >> file` in **Windows PowerShell 5.1** (`powershell.exe`)
writes UTF-16LE with a byte-order mark. The bytes at the start are `FF FE`; the TOML/JSON parser
sees `ÿ` and stops. The upstream Hugo quick start warns only that "PowerShell and Windows
PowerShell are different applications" — it never says why, which is why this costs hours.
**PowerShell 7** (`pwsh`) writes UTF-8 and does not have the problem.

**Fix.** Write files with a tool that controls encoding (an editor, or `Set-Content -Encoding utf8`
in pwsh 7; never `>` / `>>` in 5.1). Then verify the **bytes**, not the rendered text:

```powershell
Format-Hex hugo.toml | Select-Object -First 1   # must not start with FF FE or EF BB BF
```

Same trap in reverse for **content** files: a UTF-8 BOM in Markdown is tolerated by Hugo but
pollutes the first heading; keep content BOM-less.

**Status:** observed (Windows PowerShell 5.1 on Windows 11, Hugo 0.167).

## G24 — the Markdown export silently diverges from the HTML page

**Symptom.** The site renders a metadata panel above the body, but the page's `.md` output (the
machine-readable route) does not contain it — or contains it *and* the navigation around it.
Agents then read a different document from the one humans read, and neither side notices.

**Cause.** HTML and flow through different templates. Anything added only to a page-kind HTML
template (`single.html`) or only to an output-format template (`single.md.md`) exists for one
audience only.

**Fix.** Keep the two templates reading the **same data source** (page params) through shared
partials — one partial per rendering mode, both called from their template — and treat parity as
part of the definition of done. Two checks that catch the common failures:

- Regenerate `--renderToMemory` and read both artifacts for one page that has the feature
  (`public/<path>/index.html` and `public/<path>/index.md`), rather than re-reading your source.
- Notice where the two differ *deliberately*: Markdown should carry the facts and drop the
  chrome (nav, sidebar, headings you already know).

**Status:** observed (this site's teaching layer; the divergence was real before the shared-partial
rule).

**Documented counterpart.** Hugo's Markdown output format is configured with
`isPlainText = true` so the body is parsed by `text/template` rather than `html/template`
(<https://gohugo.io/configuration/output-formats/>) — without it, Markdown comes out
HTML-escaped, which is the same class of silent divergence.

## G25 — `site.Data` is deprecated and fails a warning-strict build

**Symptom.** A build that used to pass fails only when run with `--panicOnWarning`:

```text
WARN  deprecated: .Site.Data was deprecated in Hugo v0.156.0 and will be removed in a future release. Use hugo.Data instead.
```

**Cause.** The accessor moved from `site.Data` / `.Site.Data` to the global `hugo.Data`
(Hugo 0.156.0). Reading data files through the old accessor still works, so nothing breaks until
the warning is made fatal.

**Fix.** Use `hugo.Data` with `index` for a dashed filename —
`{{ index hugo.Data "glossary-alias" }}` reads `data/glossary-alias.toml`. Worth knowing: a
*shorter* path is not always the newer one, so re-check every accessor against the current version
before treating a warning as noise. This is exactly the class of change `--panicOnWarning` exists
to surface: the site had been passing `--ignoreCache` builds with the deprecated key for months.

**Status:** observed (Hugo 0.167.0; surfaced by `--panicOnWarning`, fixed the same day).

## G26 — `--printUnusedTemplates` reports a partial that is used

**Symptom.** A build with `--printUnusedTemplates` claims a partial that templates clearly call is
unused:

```text
WARN  Template /_partials/md-body.html is unused, source "…/layouts/partials/md-body.html"
```

**Cause.** The report is about *reachability within the currently rendered outputs*. A partial
called only from an output-format template that is itself conditionally exercised (here the
Markdown output templates, which also referenced the then-deprecated `site.Data`) can be reported
while the real problem is elsewhere. In this case fixing the deprecation (G25) made the warning
disappear, so the warning was a **symptom of the other defect**, not an unused file.

**Fix.** Do not delete the "unused" template on the strength of this flag. First make the build
warning-clean (`--panicOnWarning`), then re-run `--printUnusedTemplates`; only a template still
reported on a clean build is a genuine candidate. Never delete a template whose call site you can
point at.

**Status:** observed (Hugo 0.167.0; the warning vanished when G25 was fixed).

## G27 — `.Inner` is raw Markdown under *both* notations

**Symptom.** A shortcode template that outputs `{{ .Inner }}` behaves differently depending on the
delimiters, and the usual explanation ("Markdown notation hands you rendered HTML") does not match
what a dump of `.Inner` shows. Related confusion: a Markdown-notation call whose output is wrapped
in a `<div>` appears *not* to render its inner Markdown, while the same content outside the wrapper
renders fine.

**Cause.** The Markdown renderer runs on the shortcode's **output**, after the template, for
`{{% %}}` calls. `.Inner` itself is the raw inner text either way.

**Evidence (minimal site, template `<pre>[{{ .Inner }}]</pre>`).** Both calls

```text
{{% dump %}}
We design the **best** widgets in the world.
{{% /dump %}}
```

```text
{{< dump >}}
We design the **best** widgets in the world.
{{< /dump >}}
```

produce `<pre>[ We design the **best** widgets in the world. ]</pre>` — byte-identical. Only when the
template emits `.Inner` *outside* an HTML block does the Markdown-notation version end up rendered,
because the page's renderer then sees it.

**Consequence.** The familiar rule is still correct but for a different reason: Markdown notation →
do **not** also call `RenderString` (double rendering); standard notation → you **must**. And a
block-level wrapper is a trap: `<div class="stage">{{ .Inner }}</div>` starts a raw HTML block, so
the inner Markdown is swallowed unless a blank line ends the block first. That is why a "show the
live result" wrapper shortcode can only be used around shortcode calls or finished HTML, never
around Markdown prose.

**Fix.** Choose by what the *output* needs, not by what you believe `.Inner` contains; keep wrapper
templates free of Markdown, and give a Markdown-notation template a blank line after its opening
tag when the inner content must be rendered.

**Status:** observed (Hugo 0.167.0, Windows, minimal site; `<pre>` dump plus two live notations on a
documentation page).

## G28 — a remote-fetching shortcode breaks an offline, warning-strict build

**Symptom.** `hugo` exits 0 and the page silently has a hole where the shortcode was; the same
command with `--panicOnWarning` exits 2.

```text
WARN  The "x" shortcode was unable to retrieve the remote data: … error calling GetRemote: … See "content/example.md:7:1"
```

**Cause.** Hugo's `x` shortcode (and the removed `gist`) resolve their markup at build time through
`resources.GetRemote`. No network — or a host that fails the default `[security.http]` policy —
turns that into a warning, not an error, and the location renders empty. Nothing in the page or the
exit code of a plain build reveals the loss.

**Fix.** Treat "builds offline" as a design decision and check it against the content: if the site
promises an offline build, do not call remote-fetching shortcodes from content — document them, or
gate them behind a separate, network-enabled build. Note that the default policy also rejects
non-public address ranges, so a hostname resolving to a private IP fails even with working
connectivity; the warning text names the policy and the offending IP, which is often what saves the
next person an hour.

**Status:** observed (Hugo 0.167.0, Windows; a real `{{< x … >}}` call: plain build exit 0,
`--panicOnWarning` exit 2).
