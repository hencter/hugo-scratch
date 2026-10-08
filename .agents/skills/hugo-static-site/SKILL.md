---
name: hugo-static-site
description: Build, update, and verify Hugo static sites — content, front matter, sections, menus, taxonomies, i18n, themes and multi-theme layering, SEO head output and structured data, version control with git-backed dates, templates, and config keys. Use when a Hugo project or its content is an input or a deliverable, when adding or translating pages in bulk, when styling or theming a site, when wiring a repository or page dates, or when a Hugo build fails. Load this skill before editing content or templates and before running hugo.
---

# Hugo static sites

Managing a Hugo site safely means knowing which edits fail **the whole build** rather than one page,
and which failures Hugo reports against the wrong file. This skill encodes both, plus the
build/verify loop that catches them.

Resource base for this skill is `<skill-directory>`: the trap catalogue at
`<skill-directory>/references/gotchas.md`, version-keyed changes at
`<skill-directory>/references/versions.md`, layout/front-matter conventions at
`<skill-directory>/references/site-structure.md`, the SEO checklist at
`<skill-directory>/references/seo.md`, version control and dates at
`<skill-directory>/references/versioning.md` and `<skill-directory>/references/dates.md`, shortcode
authoring at `<skill-directory>/references/shortcodes.md`, optional multilingual setup at
`<skill-directory>/references/i18n.md`, the human/machine documentation parity contract at
`<skill-directory>/references/teaching-layer.md`, and the command notes at `<skill-directory>/references/commands.md`. The command notes prescribe only the
flags this workflow relies on and point at `hugo gen doc` for the reference itself — they are not a
transcription of it.

## Sources and the citation rule

Claims in this skill come from one of three places, and every reference file must say which:

- **documented** — cite the public page, e.g. <https://gohugo.io/content-management/syntax-highlighting/#escaping>.
  Never cite a path inside someone's checkout: a distributed skill cannot rely on a local clone,
  and `hugoDocs/…` paths mean nothing to another user.
- **observed** — say so, and say how (a failing build, a missing `public/` page, rendered HTML).
- **not documented** — say that too. Undocumented behaviour is worth recording *as long as it is
  labelled*, because the next person needs to know how far to trust it.

Better still: **generate instead of citing.** Anything the binary can produce (`hugo config`,
`hugo gen doc`, `hugo gen chromastyles`, `hugo list all`) needs no citation at all — the reader
regenerates it for their own version. Cite only what cannot be regenerated.

## Before touching anything

1. `hugo version` — the version decides which config keys and command names exist. Report it.
2. Check whether a `hugo server` is running. Never edit files while a watcher builds them: a
   half-written file produces a real-looking error against a file that is already fine
   (gotchas G13). If a build log looks impossible, suspect the log, not the file.
3. Establish a baseline: one-shot `hugo --ignoreCache` in the site root. Fix nothing until you
   know whether the site built before your change.

### When there is no shell

None of the three steps above can run in an environment without a shell or without the `hugo`
binary. Say so rather than pretending:

- **No `hugo version`.** Infer the applicable behaviour from the project itself — a `locale` key
  implies 0.158+, a leftover `languageCode` key implies older — and label the conclusion as
  inferred, not observed.
- **No build.** An existing `public/` tree is the output of the last successful build. Rendered
  heading ids, generated links, and which pages exist at all can be read from it; that is how a
  page that never rendered is detected (G16).
- **No file moves.** Write the new file yourself, then hand the user the exact paths to remove or
  rename. Never describe a move as done.
- **Any search tool replaces the two `grep` commands below** — the patterns matter, not the tool.
- Close by stating which checks ran and which could not, so the reader can weigh the result.

## Iron rules (these fail the entire build, not one page)

1. **Never leave an unescaped `{{<` or `{{%` in content — not even inside a fenced code block.**
   Hugo extracts shortcodes from the raw content before Markdown runs, and the documentation never
   exempts code blocks: its escaping rule (<https://gohugo.io/content-management/syntax-highlighting/#escaping>)
   is demonstrated *inside* a fenced example, every shortcode call in the docs is escaped, and the
   contributor guide (<https://gohugo.io/contribute/documentation/#escaping-shortcode-syntax>)
   gives the nested form for showing the escape itself. Write `{{</* name */>}}`,
   `{{%/* name */%}}`, closing forms `{{</* /name */>}}`, and
   `{{</*/* name */*/>}}` / `{{%/*/* name */*/%}}` when the example must show the escape.
2. **Never write the literal string `HAHAHUGOSHORTCODE` in content.** It is Hugo's shortcode
   placeholder prefix; content containing it aborts rendering with
   `illegal state in content; shortcode token missing end delim`, and Hugo attributes the error
   to whichever page is rendering. Break it as `H&#xfeff;AHAHUGOSHORTCODE` (zero-width U+FEFF via
   entity, outside any code span) when you must display it.
3. **Do not invent front-matter keys or option names.** Page build options live under `build`
   (not `_build`), where `list`/`render` take strings (`always`/`local`/`never`,
   `always`/`link`/`never`) and only `publishResources` is boolean.
4. **Never state a command, flag, config key, default, or version from memory.** Look it up with
   `hugo <command> --help`, in `hugo config` output, or in `hugo gen doc --dir <dir>` for the full
   CLI reference of the installed version; `references/commands.md` lists only the flags this
   workflow relies on. If the documentation does not say it, write "not documented" instead of
   asserting it — an invented flag or default costs a failed build and a wrong fix.

## Generate, do not transcribe

Hugo generates its own documentation from the binary. Anything you would otherwise copy by hand —
a CLI reference, the settings table, a highlight stylesheet — has a command that produces it for
the exact version installed, and generated output cannot drift:

| Need | Command | Public reference |
| --- | --- | --- |
| CLI reference, one Markdown file per command with front matter | `hugo gen doc --dir <dir>` | <https://gohugo.io/commands/hugo_gen_doc/> |
| Effective settings, defaults included (`--format`, `--printZero`) | `hugo config` | <https://gohugo.io/commands/hugo_config/> |
| Chroma stylesheet for the current highlighter | `hugo gen chromastyles` | <https://gohugo.io/commands/hugo_gen_chromastyles/> |
| Content inventory (what Hugo thinks it has) | `hugo list all` | <https://gohugo.io/commands/hugo_list_all/> |

So a reference file here does only two things: it indexes *where the generated source is*, and it
records what generation cannot give you — the traps in `references/gotchas.md` and the workflow in
this file. Note also that the documentation's own command pages are `hugo gen doc` output; they are
a snapshot of someone else's Hugo, not the authority for the version you have installed.

## Themes and layering

Create themes with the command, never by hand:

```bash
hugo new theme <name>        # → themes/<name>/ with a working skeleton (current template system)
```

A site composes themes; precedence is left to right, and the project always wins
(<https://gohugo.io/hugo-modules/theme-components/>):

```toml
theme = ["hugo-docs-theme-zh", "hugo-docs-theme"]
```

Merge rules to know before moving a file into a theme:

- `layouts`, `static`, `archetypes` merge at **file level**: the left-most file wins and files do
  not merge internally. A same-path stylesheet in an overlay theme *replaces* the base one — keep
  overlay CSS in its own file and link both.
- `i18n` and `data` merge **deeply** by key.
- A theme configuration can set only `params`, `menu`, `outputformats`, `mediatypes`. Everything
  else in a theme's `hugo.toml` is ignored — but a scaffolded theme ships demo `[menus]` entries,
  and menus *do* merge, so delete them or they appear in your navigation.

Layering that survives contact with a real site:

- project `layouts/` = **the contract only**: `baseof.html`, defining the blocks and naming the
  partials every theme must provide. Nothing else belongs there.
- the base theme = page-kind templates (`home.html`, `page.html`, `section.html`), the partials they
  call, and the stylesheets.
- an overlay theme = one concern, e.g. CJK typography: its own `assets/css/cjk.css` plus a
  `[params]` switch the base theme reads. Guard that read, or a missing param table errors:
  `{{ $on := false }}{{ with site.Params.cjk }}{{ $on = .enabled | default false }}{{ end }}`.

## Shortcodes

Create a shortcode at `layouts/_shortcodes/<name>.html` — a theme's copy participates in the same
lookup, so a site can override one shortcode; subdirectories namespace the name
(`media/audio.html` → `{{</* media/audio */>}}`).

The decision that shapes everything else is the notation, because it fixes rendering order
(<https://gohugo.io/content-management/shortcodes/#notation>):

- **Markdown notation** (`{{% … %}}`) runs *before* the Markdown renderer: `.Inner` is raw Markdown
  and its headings reach `.TableOfContents`.
- **Standard notation** (`{{< … >}}`) runs *after* it: `.Inner` is unrendered text — pipe it through
  `markdownify` — and its headings never reach the table of contents.
- Nested shortcodes render inside-out; the parent receives its children's rendered output as
  `.Inner`, and a child reaches the parent through `.Parent`.

Workflow for a request: decide the output and whether it still needs Markdown → decide arguments
(named vs positional, `.Get` / `.IsNamedParams`) → decide whether it wraps content (`.Inner`) →
write the template → call it from one page → build and read the generated HTML → document the call
syntax where it is used.

Full guide, method list, nesting, the render-hook comparison, and a verified example:
`references/shortcodes.md`. And keep the iron rule: shortcode syntax shown inside content must be
escaped.

## SEO

Search engines read template output, so SEO is a theme concern rather than a content concern. The
full checklist, the structured-data pitfall and the verification commands are in
`references/seo.md`. The three that break most often:

1. Canonical and Open Graph URLs must be absolute — `.Permalink` or `absURL`, never a relative
   path, and `baseURL` must be the real production origin.
2. Emit `noindex, nofollow` whenever `hugo.Environment` is not `production`, so previews never
   reach the index.
3. Hand structured data to the template as an **object**. `{{ $data | jsonify }}` inside a
   `<script>` is encoded as a string literal and every consumer rejects it.

## The fast loop

Content is cheap to rebuild; a wrong template is expensive to debug. Keep the loop tight:

```bash
hugo --ignoreCache          # ~0.2 s for a 200-page site: run it after every batch
hugo list all               # did the page count move the way the edit implies?
```

- Batch content edits, build once, then verify by reading the generated artifact in `public/` — not
  by re-reading your own source.
- Never hand-edit `public/`; it is output.
- Let commands answer questions (`hugo config`, `hugo list all`, `hugo gen doc`) instead of
  guessing, and keep `hugo server` for what no assertion can check.

## Version control and dates

Track source, never output: ignore `public/`, `resources/`, `.hugo_build.lock`. Then let Hugo read
the repository back, so "last updated" is a fact instead of a field that ages badly:

```toml
enableGitInfo = true

[frontmatter]
  lastmod = [':git', 'lastmod', 'date']
```

Every page then exposes `.GitInfo` (commit hash, author, subject), and templates can show the
commit the reader is actually looking at. Working rules: `references/versioning.md`.

Dates are the other half — three fields with fallback chains, a time zone, and a localization layer
that is incomplete. The essentials:

- Guard every date with `.IsZero`: a `time.Time` is a struct, so `{{ with .Date }}` never fails and
  an undated page happily prints `0001-01-01`.
- Always print machine-readable dates as `<time datetime="…">` holding ISO-8601, whatever the
  visible text says.
- Localized `:date_*` tokens fall back to English for locales the data does not cover (observed:
  `de-DE` localizes, `zh-CN` does not). Supply an explicit layout — put it in a locale overlay
  theme's `[params] dateFormat`.
- A future or expired `date` keeps a page out of the build unless you pass `--buildFuture` /
  `--buildExpired`; `hugo list future` and `hugo list expired` answer where a page went.

Full checklist: `references/dates.md`.

## Content and front matter

- Use one front-matter contract per site and keep it: `title`, `linkTitle`, `description`,
  `date`, `weight`, `source` (or whatever the site already uses). `weight` orders the sidebar,
  the section list, and prev/next; duplicate weights inside one section are a defect.
- Body starts at `##`. A page-level `#` heading duplicates the template's `<h1>`.
- Fenced code blocks carry a language tag; nested fences use four backticks outside three.
- Head slugs: Hugo keeps CJK characters, lowercases Latin, and **strips punctuation** —
  `## 草稿、将来与过期内容` becomes `#草稿将来与过期内容`. Verify an anchor by reading the
  rendered page or the generated id, never by guessing.
- Internal links are root-relative (`/section/page/`). When a page moves, every inbound link is
  now wrong — grep for the old path and migrate all of them in one pass (gotchas G15).
- Deleting a page? Check inbound links first, then delete. With no shell available you cannot
  move files, so write the new path and leave the old one for the user to remove.
- **Every scalar above the first `[table]` header.** A bare key written below a table silently
  becomes part of it (G21); the failure is invisible until the field is missing from the output.

## Human and machine documentation parity

When one site serves both readers and agents, the two must not drift apart. The working pattern —
used by the Hugo Chinese docs site this skill was distilled from — keeps a **single data source in
page params**, rendered by one partial per output format and called from both templates:

- the HTML page-kind template calls the panel partial; the Markdown output-format template calls
  the Markdown partial; both read `.Params.<feature>`;
- the facts a reader needs to judge a page (difficulty, time, prerequisites, outcomes, what to read
  next) live in front matter, not in prose, so an agent consumes them without parsing HTML;
- the machine-readable index (`pages.json` or equivalent) carries the same per-page role and the
  same values, so an agent can decide *how to use* a page — follow it, or look something up —
  without fetching it first.

Treat parity as part of done, not a follow-up: silent divergence between `.html` and `.md` is G24,
and `isPlainText = true` on the Markdown output format is the documented half of getting it right.
Full contract, field schema and the audit script: `references/teaching-layer.md`.

## Site structure and navigation

- `content/` maps to URLs: `content/a/_index.md` → `/a/`, `content/a/b.md` → `/a/b/`,
  `content/_index.md` → `/`. `index.md` (leaf bundle) and `_index.md` (branch bundle) are
  different pages; a directory cannot hold both.
- Section order comes from `weight` in each section's `_index.md`. Assign them from one scheme
  (10, 20, 30 …) and keep them unique, or the sidebar reshuffles unpredictably.
- Build the sidebar from `site.Home.Sections.ByWeight` and expand **only the current section**;
  with dozens of sections, expanding everything makes the nav unusable.
- Hugo generates `tags/` and `categories/` pages from the default taxonomies even when empty.
  Either use them or set `disableKinds = ["taxonomy", "term"]`.
- Menu entries: `pageRef` associates a page (needed by `IsMenuCurrent`/`HasMenuCurrent` and menu
  templates); `url` is for external targets. If your header only reads `.URL`, both work — but
  prefer `.Page`-aware templates when you render active states.

## Templates

- `resources.Get` returns nil for a missing file; a nil piped into `minify`/`fingerprint` aborts
  the build. Guard it: `{{ with resources.Get "css/x.css" }}{{ $r := . | minify | fingerprint }} … {{ else }}{{ errorf "missing assets/css/x.css" }}{{ end }}`.
- Go template `and`/`or`/`eq` are functions: **every argument is evaluated**. `and $p $p.Title`
  still panics when `$p` is nil. Guard with `with`, or precompute a safe string.
- `.Date`/`.Lastmod` are structs, so `{{ with .Date }}` is always true. Test `.Date.IsZero`.
- `.CurrentSection` is never nil for home/section/taxonomy/term pages, but `.Parent` may be.
- Version-sensitive: `site.Language.Lang` is deprecated (use `.Locale`); hook templates live in
  `layouts/_markup/`, partials in `layouts/_partials/`, shortcode templates in
  `layouts/_shortcodes/` in the current template system. The classic layout — `layouts/index.html`,
  `layouts/404.html`, `layouts/_default/{baseof,single,list}.html`, `layouts/partials/` — still
  works and is what many theme-free sites use. Pick one convention for a project and do not mix
  them; `references/site-structure.md` shows the classic tree and the name mapping.
- Add `layouts/404.html` (`{{ define "main" }}…{{ end }}`) if you want a real 404 page.

## Configuration keys

- `locale` (0.158+) replaced `languageCode`. Older Hugo needs the old key; do not ship both.
- `hasCJKLanguage = true` makes word counts and summaries behave for Chinese/Japanese/Korean.
- `[markup.goldmark.renderer] unsafe = true` allows raw HTML in Markdown; without it, HTML is
  stripped and replaced by a comment.
- `[markup.highlight] noClasses = false` switches Chroma to class-based output, which then
  requires your own highlight stylesheet.
- `timeZone`, `summaryLength`, `enableRobotsTXT`, `pagination.pagerSize`, `taxonomies`,
  `cascade` — check names against `hugo config` output for the installed version.

## Build and verify (Hugo's own commands — no extra tooling)

Hugo already ships the checks. Prefer them over any bespoke script: they cannot drift from the
binary you are running, and every flag below is documented in the CLI reference.

```bash
hugo version          # which behaviour applies   → https://gohugo.io/commands/hugo_version/
hugo config           # effective config          → https://gohugo.io/commands/hugo_config/
hugo list all         # content inventory         → https://gohugo.io/commands/hugo_list_all/

# one-shot build with the strict checks (flags documented at https://gohugo.io/commands/hugo/)
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates \
     --printI18nWarnings --templateMetrics --logLevel info
```

- `--printPathWarnings` reports duplicate target paths — the collision class that silently
  overwrites output; `--printUnusedTemplates` finds templates nothing reaches;
  `--printI18nWarnings` lists missing translations; `--panicOnWarning` turns the first WARNING
  into a failure, which is what stops deprecation notices from being scrolled past;
  `--ignoreCache` rules out a stale file cache when a result makes no sense.
- `hugo server -D` is a watcher: never treat a log captured while files are still being written as
  evidence (G13), and never use it as the only check.
- Before publishing, run the site audit the documentation prescribes
  (<https://gohugo.io/troubleshooting/audit/>): build with
  `HUGO_MINIFY_TDEWOLFF_HTML_KEEPCOMMENTS=true` and
  `HUGO_ENABLEMISSINGTRANSLATIONPLACEHOLDERS=true`, then grep `public/` for the documented
  patterns.

Two source-level greps cover what no flag reports. They are plain commands, not a tool to
maintain:

```bash
grep -rnE '\{\{<[^/]|\{\{%[^/]' content/   # unescaped shortcode delimiters (G1)
grep -rn 'HAHAHUGOSHORTCODE' content/      # placeholder literal (G2)
```

What still needs judgement:

1. Compare the page count Hugo prints with the number of content pages you expect.
2. `public/` is the source of truth for "did this page render": a missing
   `public/section/page/index.html` means that page never rendered, even when the console said
   nothing useful (G16).
3. Warnings are part of the result. With `--panicOnWarning` they are part of the exit status.

## Diagnosing a failing build

1. Read the error for the **page path** Hugo names. Treat the attribution as a hint, not a fact.
2. Grep that page for `{{<`/`{{%` and for `HAHAHUGOSHORTCODE`.
3. If the page is clean, bisect: move it out of `content/`, rebuild with `--ignoreCache`. If the
   build passes, the page (or its path) is the trigger; if the error moves to another file,
   Hugo misattributed it.
4. If a file is clean but keeps failing, rewrite it wholesale. A full overwrite also clears
   invisible bytes that reads and greps will not show you.
5. Report what you proved, not what you assumed: which command ran, what it printed, and which
   files changed.

## Delivery checklist

- [ ] `hugo --ignoreCache` exits 0 and the page count matches the content tree.
- [ ] The strict build above exits 0 with no warnings (or every warning is explained).
- [ ] Every page you changed has a rendered counterpart in `public/`.
- [ ] No inbound link still points at an old path.
- [ ] Any claim about defaults/versions cites the source you verified it against.
- [ ] If no shell was available: the report says which checks ran and which could not, and no
      command output is presented as if it had been observed.
