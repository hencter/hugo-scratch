+++
title = 'Content'
linkTitle = 'Content'
description = "How the content directory maps to URLs, and who owns a page's front matter, body and resources."
weight = 30
+++

This chapter is about the `content/` directory itself: how a Markdown file has to be written and where it has to live before it becomes the right URL — and before the templates, the search index and the machine-readable outputs all read the right things out of it.

## Four pages, one job each

The chapter is short, but the order is not arbitrary:

- **front matter** decides a page's identity — its title, date, weight, and the extra fields this theme actually reads;
- **page bundles** decide where resources live, and why `content/blog/hugo-pipes/cover.png` has to sit next to `index.md`;
- shortcodes and render hooks cover everything Markdown cannot do on its own, and they are documented separately in [shortcodes](/docs/content/shortcodes/) and [render hooks](/docs/content/render-hooks/).

None of the four is abstract: every one of them points at a file that exists in this repository.

## The site has exactly one content contract

There are no per-directory front matter rules in `config/_default/hugo.toml`. Every page uses the same fields — `title`, `linkTitle`, `description`, `date`, `weight` — and documentation pages add `difficulty`, `estimatedTime`, `prerequisites` and `outcomes` on top. Those four are not decoration: `layouts/_partials/facts.html` turns them into the panel at the top of the page, and `layouts/home.pages.json` writes the same values into the machine-readable index.

{{< note >}}
Keep the facts in front matter and the panel and `pages.json` cannot disagree. Move "about 15 minutes" into the prose instead, and one of the two outputs is now wrong.
{{< /note >}}

## From `content/` to a URL

| File | URL | Kind |
| --- | --- | --- |
| `content/_index.md` | `/` | Home page |
| `content/docs/_index.md` | `/docs/` | Section index (branch bundle) |
| `content/docs/content/front-matter.md` | `/docs/content/front-matter/` | Regular page |
| `content/blog/hugo-pipes/index.md` | `/blog/hugo-pipes/` | Leaf bundle |
| `content/about/index.md` | `/about/` | Leaf bundle |

Two rules cover every case. `_index.md` produces a section page and any other `.md` produces a regular page; the file name is the slug, with `index.md` as the exception that turns its own directory into the page.

## Bilingual pages pair up by file name

The Simplified-Chinese text lives in `front-matter.md` and the English text in `front-matter.en.md`, in the same directory. Hugo pairs files that share a base name and a directory plus a language suffix, so `.Translations`, `.AllTranslations`, the language switcher and the `xhtml:link` cross-references in `sitemap.xml` all work without splitting `content/` into two parallel trees.

Pairing only holds if the two files share a shape: the same heading structure, the same shortcode calls, the same code blocks. They are not two documents — they are one page in two languages.

## Where to go next

If you have just cloned the repository and only want to change a sentence, start with [quick start](/docs/start/quick-start/). If you are adding a page and want it to appear in the sidebar, read [front matter](/docs/content/front-matter/) first: `weight` decides where the page lands in the sidebar and in the previous/next navigation. Get it wrong and the page does not disappear — it just moves.
