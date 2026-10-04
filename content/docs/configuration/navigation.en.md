+++
title = 'Navigation'
linkTitle = 'Navigation'
description = 'pageRef versus url, where the sidebar and breadcrumbs read from, how pagination is sized, and why menu labels stay out of i18n.'
date = 2026-01-14
weight = 20
difficulty = 'intermediate'
estimatedTime = 18
prerequisites = ['/docs/configuration/site-config/']
outcomes = ['Choose between pageRef and url deliberately', 'Know which config drives the sidebar, breadcrumbs and page links', 'Put menu labels in the right place']
tags = ['Hugo', 'navigation']
+++

Navigation is the first thing a reader sees and the part most likely to break when the site is localised. Four independent mechanisms are involved — the header menu, the documentation sidebar, breadcrumbs and previous/next links — along with list pagination and the taxonomies underneath. They read different configuration and use different templates, and treating them as one topic only makes them harder to reason about. So, one at a time.

## Menus are data, not strings in a template

Both the main menu and the footer menu are defined in the per-language menu files under `config/_default/`: `menus.zh-cn.toml` and `menus.en.toml`. Every entry needs at least a `name` and a `weight`, plus one way of pointing at a target. `layouts/_partials/menu.html` does the rendering: it sorts by `weight` and walks children recursively, and the header and the footer share that one template, differing only in the menu ID they are passed.

The most valuable line in an entry is `weight`. Menu order comes entirely from it; the order the entries appear in the file plays no part. Reordering the menu means changing numbers, not moving blocks of text.

## pageRef versus url

Use `pageRef` for pages inside the site and `url` for destinations outside it. The main menu uses both:

```toml
[[main]]
  name = '文档'
  pageRef = '/docs'
  weight = 10

[[main]]
  name = 'Hugo 官网'
  url = 'https://gohugo.io/'
  weight = 90
```

The difference is not merely internal versus external; it is whether the entry can take part in current-page detection. Highlighting the current menu entry means answering two different questions, and `layouts/_partials/menu.html` asks both:

- `IsMenuCurrent` — the current page **is** this entry. On a match the template adds the `is-current` class and `aria-current="page"`, so a screen reader announces the entry as the current page.
- `HasMenuCurrent` — the current page sits **below** this entry. On a match it adds `is-ancestor`, which is why "Docs" in the header stays lit while you read `/docs/configuration/navigation/`.

Both checks require the entry to be associated with a page object, which means they require `pageRef`. That brings a second benefit: the menu follows the page when its URL changes, so renaming `/docs/` to `/documentation/` needs no edit here. An external entry written with `url` has no page to compare against, so the template treats it as an external link and skips both checks. That is also why the template tests `strings.HasPrefix .URL "http"` before testing the current entry: reverse the order and external links get compared as if they were internal.

{{< warning >}}
`pageRef` takes a content path rather than a final URL, but it must point at a page that exists. Get it wrong and Hugo will not stop the build; the menu simply finds no page object, so both the active highlight and the ancestor check fail at once. The symptom is a menu that works but never highlights the current page.
{{< /warning >}}

## Why menu labels stay out of i18n

Interface strings go through the `i18n/` directory — "previous", "next", "table of contents". Menu labels do not, for two reasons, the second more important than the first:

- A menu label is navigation, not a string inside a template. Putting it in the translation catalogue means every adjustment to the navigation also has to touch the translation files.
- A single missing translation key makes Hugo emit a `MISSING_TRANSLATION` warning on **every page**. There are only a handful of menu entries, and carrying the risk of a build that fills the log with warnings on their behalf is a poor trade.

More to the point, an English site usually does not translate its menu entry for entry; it replaces or reorders them. Two languages can legitimately have different menu structures. With one file per language, `menus.en.toml` can have its own order and its own entries instead of mirroring the Chinese one.

## Sidebar, breadcrumbs and page links

All three are switched in `config/_default/params.toml`, and all three behave in templates:

- **The sidebar** appears according to `[params.nav] sidebarSections = ['docs']`, which is why the documentation section has in-section navigation and the blog does not. `layouts/_partials/sidebar.html` takes the root of the current page's section and walks `.Pages.ByWeight` — sections and regular pages sort together in one list, so an interleaved order of "section landing page, a few pages, a subsection" stays stable. Only the branch holding the current page is expanded; the rest stay collapsed.
- **Breadcrumbs** are controlled by `[params.ui] showBreadcrumbs`, and the template walks `.Ancestors.Reverse` from the home page down to the current one. The same trail is also written into the document head as `BreadcrumbList` structured data, and the two must agree: a visible breadcrumb that contradicts its structured data is an error, not a cosmetic detail.
- **Previous / next** are controlled by `[params.ui] showPrevNext` and use `.PrevInSection` and `.NextInSection`. They follow the section's own ordering (weight first, then date), which is the same source the sidebar uses, so paging and the sidebar can never disagree.

## Pagination

The page size for list pages is set by `[pagination] pagerSize = 2` in `config/_default/hugo.toml`. That value is deliberately small so the pagination control actually appears on a demo site: `/blog/` holds two posts, so the section paginates immediately. A real site wants 10 or more.

The template side is worth remembering: `layouts/section.html` uses `.Paginator`, not `.Paginate`. The reason is that `head/meta.html` has already called `.Paginator` to build the paginated `<title>` and the canonical URL. Calling `.Paginate` again on the same page with a different collection would produce two conflicting paginators. The rendering itself is `layouts/_partials/pagination.html`, which draws first page, last page and a window around the current page once there are more than eight, using an ellipsis for each gap — so a sixty-page archive does not render sixty links.

## Taxonomies

`[taxonomies]` in `config/_default/hugo.toml` declares three: `tag = 'tags'`, `category = 'categories'` and `series = 'series'`. Hugo generates a list page and a term page for each of them even when no content uses them. Posts opt in through front matter:

```toml
tags = ['Hugo', '主题']
categories = ['工程实践']
series = ['从零搭一个 Hugo 站点']
```

`[params.ui] showTags` decides whether a post page displays its tags. The taxonomy pages themselves are rendered by `layouts/taxonomy.html` and `layouts/term.html`, with the term list delegated to `layouts/_partials/terms.html`. If you decide not to use taxonomies at all, add `disableKinds = ['taxonomy', 'term']` to the configuration so Hugo stops generating those pages — but check first that no template or menu entry refers to them.

That is the whole of the navigation configuration. Run a build after changing it: the warnings usually say straight away that a menu entry points at a page that does not exist, or that a section's `weight` duplicates its neighbour's.
