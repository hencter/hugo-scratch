+++
title = 'Changelog'
linkTitle = 'Changelog'
description = 'Generated from data/changelog.toml by content/changelog/_content.gotmpl; both languages read the same data.'
weight = 30
+++

## One source of truth

Not one page in this section is hand-written. The entire release history lives in a
single file, `data/changelog.toml`, where each `[[releases]]` entry carries
`version`, `date`, `title` / `title_en` and the two arrays `notes` / `notes_en`.
Edit one entry and the lists and pages on both language sides move together.

That is also why every text field in that file comes in pairs. The Chinese and
English readers are looking at the same data rather than two documents somebody has
to keep aligned by hand. With one source of truth there is no state in which the
list has been updated and a page has not.

## What the content adapter does at build time

`content/changelog/_content.gotmpl` is a **content adapter**: a Go template that
creates pages at build time. It walks `hugo.Data.changelog.releases` and hands over
each version:

```go-html-template
{{- $slug := replace .version "." "-" -}}
{{- $notes := slice -}}
{{- range .notes }}{{ $notes = $notes | append (printf "- %s" .) }}{{ end -}}
{{- $body := printf "## %s\n\n%s\n" .title (delimit $notes "\n") -}}

{{- $.AddPage (dict
      "path"        $slug
      "kind"        "page"
      "title"       (printf "%s — %s" .version .title)
      "description" .title
      "date"        .date
      "params"      (dict "version" .version)
      "content"     (dict "mediaType" "text/markdown" "value" $body)) -}}
```

`path` is relative to this content directory and carries no extension; the dots in
`.version` are swapped for hyphens, so `v1.11.0` lands at `/changelog/v1-11-0/`.
`content` is a `mediaType` plus a `value`, which means the body is ordinary
Markdown and goes through the same rendering path as any other page — render hooks
included, and an `index.md` twin of its own.

The payoff is concrete: **the list and the pages cannot disagree**, because both are
projections of the same render, and adding a release means appending one TOML entry
instead of creating a directory and remembering to update the list.

## What it cannot do

A content adapter creates pages in the **default language** only. Giving `path` a
language suffix does not produce a translation; it produces a page whose URL
literally contains that suffix — `/changelog/v1-0-0.en/` is a page of its own, not
the English version of `/changelog/v1-0-0/`. `.Translations` is therefore empty
throughout this section, and the language switcher falls back to the home page on
these pages.

The English side takes the other road: the same data is rendered inline by the
`{{< changelog >}}` shortcode from `_index.en.md`. The reader sees the same release
history; its carrier is the English section page rather than a set of
"English-language" generated pages.

> [!WARNING]
> The `notes` arrays are Markdown per line: each string becomes one `- ` item in a
> list. Multiple lines inside a single note do not become multiple paragraphs, so
> split a thought across two entries when you need a break.

## Pagination and version numbers

`pagination.pagerSize` is **2** in `config/_default/hugo.toml`. That is far too small
for a real site; it is here so that the pagination control actually appears on this
demo. The section therefore spreads over several pages, which is also what lets the
windowed branch of `layouts/_partials/pagination.html` run.

Version numbers follow `v<major>.<minor>.<patch>`, matching the theme repository's
tags: each version corresponds to a commit in the theme and one in the site, and the
`date` in `data/changelog.toml` is the date of that commit.
