# Dates, time, and localized output

Dates look trivial and are not: three different fields, several fallbacks, a time zone, and a
localization layer that silently falls back to English for some languages.

## The three fields

| Field | Meaning | Source |
| --- | --- | --- |
| `date` | when the content was published | front matter, else the `[frontmatter].date` fallback chain |
| `lastmod` | when it last changed | front matter, else (with `enableGitInfo`) the commit date |
| `publishDate` | when it should become visible | front matter; future-dated pages need `--buildFuture` |

`[frontmatter]` decides which fields fill which property, in order
(<https://gohugo.io/configuration/all/#frontmatter>). Listing `:git` first for `lastmod` makes
"last updated" reflect real commits — see `versioning.md`.

Pitfalls:

- `time.Time` is a struct, so `{{ with .Date }}` is **always** true. Test `.Date.IsZero` instead,
  or a page without a date prints `0001-01-01`.
- A future `date` hides the page from the build until you pass `--buildFuture`, and an expired
  one hides it behind `--buildExpired`. Check `hugo list future` / `hugo list expired` rather
  than wondering where a page went.
- Set `timeZone` in the config, or a date without an offset is interpreted in the build machine's
  zone. `--clock` fixes "now" for reproducible builds.

## Formats

Go reference-time layouts are the smallest surprise: `2006-01-02` is ISO, `2006年1月2日` is the
Chinese equivalent, `2 Jan 2006` is day-first.

Hugo also ships localized tokens — `:date_full`, `:date_long`, `:date_medium`, `:date_short`, and
the `:time_*` siblings — which format according to the language
(<https://gohugo.io/functions/time/format/>). Hugo resolves the locale from the `locale`
configuration setting, falling back to the language key, and the value must be a locale known to
the underlying `bep/golocales` package.

**Observed:** the tokens do not cover every language. With `locale = "de-DE"` the same template
renders `1. Oktober 2026`; with `locale = "zh-CN"` it renders `October 1, 2026` — an English
fallback, not an error. Do not assume a CJK locale is localized just because the config looks
right: build once and read the output.

The robust pattern is a configurable layout with a sensible default:

```go-html-template
{{ $layout := site.Params.dateFormat | default ":date_long" }}
<time datetime="{{ .Date.Format "2006-01-02T15:04:05Z07:00" }}">{{ time.Format $layout .Date }}</time>
```

```toml
[params]
  dateFormat = "2006年1月2日"      # a locale overlay theme can supply this
```

Always wrap machine-readable dates in `<time datetime="…">` with an ISO-8601 value: it is what
search engines and assistive technology read, and it stays correct no matter how the visible text
is formatted.

## Relative time

Hugo has no "3 days ago" helper; compute it from a `time.Duration`:

```go-html-template
{{ $days := int (div (int (now.Sub .Lastmod).Hours) 24) }}
{{ if eq $days 0 }}今天{{ else if eq $days 1 }}昨天{{ else }}{{ $days }} 天前{{ end }}
```

Guard the sign: a page dated in the future (or built with `--clock`) yields negative days, which
should print nothing rather than "-2 天前".

## Sorting and listing

- `.Pages.ByDate`, `.ByLastmod`, `.ByPublishDate`, `.ByTitle`, `.ByWeight` are the orderings;
  `ByWeight` is the usual choice for hand-ordered documentation.
- `hugo list drafts|future|expired|published` answers "what is not in this build" without any
  template work.
- Feed and sitemap output carry dates too; if they look wrong, fix the fields rather than the
  templates.
