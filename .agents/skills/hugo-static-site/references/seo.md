# SEO for Hugo sites

Everything search engines read is produced by templates, so this is a checklist for the theme,
not for the content. Ordered by impact.

## Per-page head

| Tag | Source | Why |
| --- | --- | --- |
| `<title>` | `{{ .Title }} · {{ site.Title }}`, home = site title | one unique title per page; the strongest single signal |
| `<meta name="description">` | page `description` → `.Summary` → `site.Params.description`, then `plainify \| truncate 160` | affects click-through; must be unique, so do not let every page inherit one sentence |
| `<link rel="canonical">` | `{{ .Permalink }}` | absolute; collapses duplicate URLs |
| `<meta name="robots">` | `noindex, nofollow` when `hugo.Environment` is not `production`, or `noindex: true` in front matter | keeps `hugo server` previews and staging out of the index |
| Open Graph | `og:title`, `og:description`, `og:type` (`article`/`website`), `og:url`, `og:locale`, `og:image` | social previews; image must be an absolute URL |
| Twitter card | `summary`, or `summary_large_image` when an image exists | ditto |
| `<link rel="alternate" type="application/rss+xml">` | `{{ with .OutputFormats.Get "rss" }}` | exists only on home/section/taxonomy pages — do not expect it on a regular page |
| JSON-LD | `TechArticle` for pages, `WebSite` for home | rich results |
| `hreflang` | `.AllTranslations` — only when the site actually has > 1 language | pairing translations; a self-referential pair is noise |
| `theme-color` | `media="(prefers-color-scheme: light/dark)"` | mobile browser chrome |

## Structured data: the pitfall that costs a build

Never write `{{ $data | jsonify }}` inside `<script type="application/ld+json">`. The `<script>`
element puts the template in a JavaScript context, so Go encodes the already-serialised string as
a JS **string literal**:

```html
<script type="application/ld+json">"{\"@context\":\"https://schema.org\"}"</script>
```

That is a string, not an object — every consumer rejects it. Hand the object itself to the
template instead and let the JS context serialise it sanitised:

```go-html-template
{{ $data := dict "@context" "https://schema.org" "@type" "TechArticle" "name" .Title }}
<script type="application/ld+json">{{ $data }}</script>
```

Documented at <https://gohugo.io/functions/safe/js/>: "A safe alternative is to parse the JSON with
`transform.Unmarshal` and then pass the resultant object into the template, where it will be
converted to sanitized JSON when presented in a JavaScript context." Build the object with `dict`
and `<` is emitted as `\u003c`, so the block cannot break out of the script element.

## Site level

- `enableRobotsTXT = true` **plus** a `layouts/robots.txt` template: allow crawling only when
  `hugo.Environment` is `production`, and always print `Sitemap: {{ "sitemap.xml" | absURL }}`.
- `[sitemap]` (`changefreq`, `priority`) in the site configuration; Hugo emits `sitemap.xml`.
- `baseURL` must be the real production origin, or canonical, Open Graph, `hreflang` and the
  sitemap all advertise the wrong host.
- `[markup.tableOfContents]` with `startLevel = 2`, `endLevel = 3` keeps generated heading ids and
  the table of contents aligned with the outline.
- Remove or de-index thin pages: `disableKinds = ["taxonomy", "term"]`, or `noindex` per page.
- Exactly one `<h1>` per page — the template's. Body headings start at `##`; heading order is
  document structure, not visual sizing.
- Unique URLs: an `aliases` entry creates a redirect, a duplicate `slug` creates a collision.
  `--printPathWarnings` reports the collisions.

## Performance (ranking-adjacent)

- Fingerprint and minify CSS/JS (`resources.Get | minify | fingerprint`) and mark the tags with
  `integrity`.
- Prefer system font stacks; a webfont fetch delays first paint.
- Keep the sidecar light: a documentation site needs no third-party scripts.
- `hasCJKLanguage = true` makes summaries and word counts meaningful for Chinese/Japanese/Korean
  content instead of counting a whole sentence as one word.

## Verify with commands, not with opinion

```bash
hugo --ignoreCache                 # production → "index, follow"
hugo server -D                     # development → "noindex, nofollow"
grep -o '<link rel="canonical"[^>]*>' public/index.html
grep -l 'application/ld+json' public/**/*.html | wc -l
```

Then parse one JSON-LD block with a real JSON parser. A tag that renders but does not parse is
worse than no tag: it is an error you shipped deliberately. In PowerShell:

```powershell
$h = Get-Content public\index.html -Raw
$m = [regex]::Match($h, '<script type="application/ld\+json">(.*?)</script>', 'Singleline')
$m.Groups[1].Value | ConvertFrom-Json
```
