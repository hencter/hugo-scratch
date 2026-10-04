+++
title = 'Reference'
linkTitle = 'Reference'
description = 'The earlier chapters compressed into tables you can look things up in: feature, file, config key and template.'
weight = 90
+++

The earlier chapters are written to be read in order. This one is used the other way round: you know what you want to change, and you come back to find where it lives. So it repeats no narrative and gives only entry points that name a real file or config key.

## The files in this repository already explain themselves

Almost every config key in this theme carries its default in a comment at the top of the file, with the key name, the value and the reason in one place — better than any cheat sheet:

- `config/_default/hugo.toml` — `[outputs]`, `[frontmatter]`, `[markup]`, `[taxonomies]`, `[cascade]`, `[imaging]`, `[caches]`, `[security]`, `[privacy]`;
- `config/_default/params.toml` — UI switches, social images, verification codes, the comments repository;
- `config/_default/languages.toml` — each language site's `label`, `locale` and date format;
- `themes/hugo-scratch-theme/hugo.toml` — the four key groups a theme may declare: `params`, `menu`, `outputformats`, `mediatypes`;
- `data/changelog.toml` — the single source of truth for the changelog.

To find out whether a key exists in the Hugo you have, and what its default is, neither guess nor copy a web page: run `hugo config`, which prints the **merged** configuration.

## From "what I want to change" to a file

| Goal | Where to look |
| --- | --- |
| Add a documentation page | Create a `.md` under `content/docs/` with a `title` and a `weight`; see [front matter](/docs/content/front-matter/) |
| Give a page an image | Make it a leaf bundle and keep the image beside `index.md`; see [page bundles](/docs/content/page-bundles/) |
| Change which sections get a sidebar | `[params.nav] sidebarSections` in `config/_default/params.toml` |
| Change the navigation menus | `config/_default/menus.<lang>.toml`, associating pages with `pageRef` |
| Change styling | Your site's own `assets/css/custom.css` — do not overwrite the theme's `main.css` |
| Change interface wording | `themes/hugo-scratch-theme/i18n/<lang>.toml`, adding the key in both languages |
| Add or remove a machine-readable output | `[outputs]` in `config/_default/hugo.toml`; template names in [output formats](/docs/templates/output-formats/) |
| Change sitemap frequency | The page's `[sitemap]` table, or a site-level `[[cascade]]` |
| Swap the comments provider | Create `layouts/_partials/comments.html` in your site to override the theme's |

## Three boundaries worth memorising

{{< warning >}}
`.Site.Data`, `.Page.IsNode`, `site.Sites`, `.Language.LanguageName`, `cascade._target` and `_build` are all deprecated. The replacements are `hugo.Data`, `.IsPage` / `.IsBranch`, `hugo.Sites`, `.Language.Label`, `cascade.target` and `build`. Under `--panicOnWarning` a single deprecation warning is a failed build.
{{< /warning >}}

Two more are less visible but cost just as much time to track down:

- shortcode delimiters in content must be escaped, including inside fenced code blocks:

  ```text
  {{</* note */>}} content {{</* /note */>}}
  {{%/* tabs */%}} … {{%/* /tabs */%}}
  ```
- in templates, `and` and `or` evaluate every argument, `.Date` is a struct and therefore always truthy, and `default true` cannot express an explicit `false`. All three have runnable examples in [templates and lookup order](/docs/templates/templates/).

## Let commands replace memory

```bash
hugo version      # the version decides which keys and commands exist
hugo config       # the merged configuration, defaults included
hugo list all     # the content inventory as Hugo sees it
hugo --ignoreCache --panicOnWarning   # one warning is a failure
```

Those four answer most of "did I change the right thing". The one judgement left is whether the page exists in `public/` — the output is the truth, and a quiet console does not mean a page rendered.

For a side-by-side view of which feature lands in which file, read the [feature matrix](/docs/reference/feature-matrix/): it lays out config keys, templates, shortcodes and output formats by feature, which is what you want before a change whose blast radius is unclear.
