# Version control for a Hugo site

A Hugo site is source plus generated output; version control should hold the source and never the
output. Hugo can also read the repository back, which is what makes "last updated" trustworthy.

## What to track, what to ignore

Track: `hugo.toml`, `content/`, `layouts/`, `assets/`, `themes/` (or a `go.mod` when themes come
from modules), `archetypes/`, `data/`, `i18n/`, `static/`, and the `README`.

Ignore, because `hugo` regenerates them:

```gitignore
public/
resources/
.hugo_build.lock
```

Also ignore any large read-only reference clone you keep beside the site, and editor/OS noise.
Add a `.gitattributes` with `* text=auto eol=lf` so a mixed-platform team does not commit
whole-file line-ending changes.

## Let Hugo read the repository

```toml
enableGitInfo = true
```

With that, every page exposes `.GitInfo` (`.AbbreviatedHash`, `.Hash`, `.AuthorName`, `.Subject`,
`.CommitDate`) and `:git` becomes usable in the front-matter date mapping
(<https://gohugo.io/methods/page/gitinfo/>).

```toml
[frontmatter]
  lastmod = [':git', 'lastmod', 'date']
  date = ['date', ':git']
```

The list is an ordered fallback: the first field that resolves wins. Putting `:git` first for
`lastmod` means "last updated" is the real commit date instead of a hand-written field that ages
badly. Without a repository (or with uncommitted files) `:git` resolves to nothing and the next
entry is used, so keep `date` in the list and guard `.Lastmod.IsZero` in templates.

## Displaying provenance

A page footer that shows the commit is honest about what the reader is looking at:

```go-html-template
{{ with .GitInfo }}<span title="{{ .Subject }}（{{ .AuthorName }}）">提交 {{ .AbbreviatedHash }}</span>{{ end }}
```

Keep a human version for the site itself (`[params] version = "v1.0.0"`), bump it when you cut a
release, and tag the same point in history (`git tag -a v1.0.0 -m "…"`). The tag and the config
value are two statements of one fact; keep them in step.

## Working rules

- Commit content batches separately from template changes: a build failure then has one obvious
  suspect.
- Never commit `public/`. Publishing is a build step, not a commit.
- A deployment tool builds from a commit; make sure the working tree is clean before you rely on
  anything you just built (`git status --short`).
- If a page's "last updated" looks wrong, check whether the file is actually committed —
  `enableGitInfo` reads the repository, not the filesystem clock.
