+++
title = 'Working on this site through an agent'
linkTitle = 'Agent workflow'
description = 'Where an agent enters, where the facts live, how a change is proved, and the iron rules of content authoring.'
date = 2026-03-02
weight = 10
difficulty = 'intermediate'
estimatedTime = 15
prerequisites = ['/docs/start/directory-structure/', '/docs/']
outcomes = ['Know which file an agent reads first and where a page keeps its facts', 'Prove a change with the strict build and the rendered output', 'Tell source apart from public/ and resources/', 'Add a page-level table of contents or a banner the supported way']
tags = ['agents']
+++

## Two entry points: AGENTS.md and page front matter

`AGENTS.md` at the repository root is the only file that has to be read first. It is the brief an agent works from, and it should stay short and actionable: the directory map, the build command, the paths that are off limits, the content rules. When the structure or a rule changes, change that file — do not start a second explanation somewhere downstream, because the two will drift and an agent reads only one of them.

The second entry point is each page's own front matter. A page's cost and its payoff live in fields rather than in prose:

```toml
weight = 10
difficulty = 'intermediate'
estimatedTime = 25
prerequisites = ['/docs/start/quick-start/', '/docs/configuration/site-config/']
outcomes = ['Explain why baseURL must match the address the site is actually served from', 'Switch the repository Pages source to GitHub Actions']
```

`layouts/_partials/facts.html` renders those fields as the facts panel at the top of the page, and `/pages.json` (see [output formats](/docs/templates/output-formats/)) carries the same values, so a person and an agent are reading one answer. `prerequisites` holds root-relative paths, which the panel passes to `site.GetPage`: when the lookup succeeds it prints the target page's short title and real link, and when it fails it prints the raw string.

{{< note >}}
In TOML every scalar must appear before the first `[table]` header. A bare key written below one silently becomes a member of that table, the field vanishes from the rendered page, and nothing reports an error.
{{< /note >}}

## How an agent proves a change

One command, and then one reading of the output.

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

Exit code 0 means the build succeeded with no warnings at all — no internal link that resolves to no page (`layouts/_markup/render-link.html` warns about those), and no template that nothing reaches. But an exit code only proves nothing broke; it does not prove the change is right. The second step is reading the rendered file: `public/docs/start/quick-start/index.html` has to contain the sentence you just wrote, and the Markdown output of the same page, `public/docs/start/quick-start/index.md`, has to agree with it. To find out whether a page rendered at all, look for its directory under `public/`; do not assume it made the build just because the console stayed quiet.

{{< tip >}}
`hugo list all` prints every page Hugo believes it has. When the count is wrong, the usual cause is one page's front matter keeping it out of the build, not a broken template.
{{< /tip >}}

## Output is not source

`public/`, `resources/`, `hugo_stats.json` and the build lock `.hugo_build.lock` are output or state files, which is how `.gitignore` already treats them. They are not places where a quick edit is acceptable:

- `public/` is the build destination. The next `hugo` run rewrites it, so a hand edit is lost; worse, you would be reading a page that does not represent the source and drawing conclusions from it.
- `resources/_gen/` is the resource cache produced by Hugo Pipes. Deleting it is fine; editing it is a bet against the next build.
- `hugo_stats.json` only exists when the build is asked to write statistics. It is output for external tools, not configuration.

## Iron rules of content authoring

- **Shortcode syntax you want to show must be escaped.** Write `{{</* note */>}}` … `{{</* /note */>}}` and `{{%/* tabs */%}}` … `{{%/* /tabs */%}}`. An unescaped shortcode opening delimiter in prose (two left braces followed by `<` or `%`) is really executed, **including inside a fenced code block** — a fence does not protect you.
- **Never write Hugo's internal placeholder.** While rendering shortcodes, Hugo first replaces each call with an all-caps placeholder string and swaps the result back after Markdown has run. If that string ends up in content — most often by copying it out of a build log or an output fragment — rendering fails with `illegal state in content`, and Hugo charges the error to whichever page was rendering at the time, which points at an innocent file.
- **The body starts at `##`.** The page's `<h1>` comes from the template, so a second `#` in the body is a duplicate heading.
- **Bilingual means paired.** A change to `name.md` has to reach `name.en.md` in the same directory, with the same heading levels, the same code blocks and the same shortcode calls.
- **Internal links are root-relative** (for example `/docs/start/quick-start/`) and must resolve to a page that exists, or the strict build fails outright.

## Page-level table of contents and banners

Both needs have a supported switch. Do not write your own HTML for either:

- **In-page table of contents**: call `{{</* toc */>}}` in the body and `layouts/_shortcodes/toc.html` prints the `.TableOfContents` Hugo generated for this page; the levels it includes come from `[markup.tableOfContents]` in the configuration. It is a different implementation from the left-hand sidebar, where `layouts/_partials/toc.html` rebuilds the tree from `.Fragments` with scroll highlighting and obeys `[params.ui] tocMinHeadings`. To switch the sidebar contents off on one page, set `toc = false` in its front matter.
- **Page banner**: set `notice = 'This page is being rewritten'` in front matter and `layouts/_partials/banner.html` renders a note at the top of the page. `notice` is a literal string, not an i18n key — whatever you write is what appears. A draft also shows a second banner when the build includes drafts with `-D`.

This page calls the table-of-contents shortcode once. What follows is the result it prints at that position in the body, listing the same headings as the sidebar tree rebuilt from `.Fragments`:

{{< toc >}}

## One real request, end to end

The request: "The quick start page has to say that cloning needs `--recurse-submodules`, and it needs a note at the top."

1. **Locate.** The agent reads `AGENTS.md`, then the front matter of `content/docs/start/quick-start.md`, to establish which page changes and what that page costs a reader.
2. **Touch the files.** The change lands in `content/docs/start/quick-start.md` and `content/docs/start/quick-start.en.md`, keeping heading levels and code blocks aligned. The banner goes into the `notice` front-matter field rather than into hand-written HTML in the body. If the rule itself is missing from `AGENTS.md`, that file is updated too — it is what the next agent reads.
3. **Prove it.** Run the strict build for exit code 0; read `public/docs/start/quick-start/index.html` and confirm both the new sentence and the banner element are there; read `public/docs/start/quick-start/index.md` and confirm the Markdown output matches; then confirm with `hugo list all` that the page count is unchanged, because a content-only change must not change how many pages the site has.

## One definition of the gate

{{< include "build-gate" >}}
