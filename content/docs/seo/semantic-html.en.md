+++
title = 'Semantic HTML'
linkTitle = 'Semantic HTML'
description = 'Landmarks, the single h1, the skip link, aria-current, machine-readable dates and language attributes, and how images, tables, print and reduced motion are handled.'
date = 2026-02-25
weight = 10
difficulty = 'beginner'
estimatedTime = 12
prerequisites = ['/docs/', '/docs/seo/']
outcomes = ['Name what each landmark does in this layout', 'Know how heading levels and anchors are produced', 'Explain why a date is written as time datetime']
tags = ['SEO']
+++

Anyone writing Markdown rarely notices this layer: you type `##` and the page acquires landmarks, a skip link, current-item markers and machine-readable dates on its own. All of it comes from templates. This page goes through each piece, because every one of them serves two readers at once — a screen reader and a crawler — and each only has to be right once, in the template.

## Landmarks: who wraps whom

`layouts/baseof.html` is the contract for that division of labour. The document opens with `<html lang="…" dir="…">`, immediately followed by a skip link, then the header, the content grid and the footer. Each page-kind template fills only the `main` block; the position is decided here:

- `layouts/_partials/header.html` emits a real `<header class="site-header">` containing `<nav id="site-nav">` labelled from the `mainNavigation` translation string. The mobile disclosure button only toggles visibility; the menu itself is always in the HTML.
- `layouts/_partials/sidebar.html` and `layouts/_partials/toc.html` each emit a `<nav>`, distinguished by `aria-label` and `aria-labelledby`: one is the section navigation, the other the on-this-page contents. Three navigation landmarks on one page is perfectly normal, provided each one has a name.
- `layouts/_partials/facts.html` emits `<aside class="facts">`. Difficulty, estimated time, prerequisites and outcomes are supplementary to the body, so they are an `aside` rather than a paragraph squeezed under the title.
- `layouts/_partials/footer.html` emits `<footer class="site-footer">`, whose links to the machine-readable files serve readers and crawlers at once.

{{< note >}}
The point of a landmark is being skippable. A screen-reader user should not have to hear the navigation again on every page and can jump straight to `main`; a crawler uses the same boundaries to decide which part is the body. That is why `<div class="header">` and a real `<header>` look identical and are not interchangeable.
{{< /note >}}

## One h1 per page, and no skipped levels

The `h1` comes from the page template: `layouts/page.html` and `layouts/section.html` both put it in `page__header` and fill it from `.Title`. Markdown body text must therefore start at `##` — another `#` would produce two top-level headings and break the outline.

Body headings are rendered by `layouts/_markup/render-heading.html`, which does three things: it keeps the `.Anchor` Hugo computed as the `id` while letting `{#custom-id}` override it; it passes block attributes such as `{.class}` through to the element; and it appends a self-referencing anchor link carrying an `aria-label`, so it is a real link with an accessible name rather than a decorative `#` character.

How deep the table of contents goes is set by `[markup.tableOfContents]` in `config/_default/hugo.toml`, currently `startLevel = 2` and `endLevel = 3` — exactly the two levels body text may use. `layouts/_partials/layout/flags.html` also counts the headings and skips the sidebar contents entirely below three of them.

## The skip link and the current item

The skip link is the first stop for a keyboard user. It is the first element in `baseof.html`, it targets `#main`, and the `tabindex="-1"` on `<main id="main" …>` is part of the mechanism: without it, focus cannot be moved to `main` by a link. `.skip-link` in `assets/css/base.css` sits outside the viewport until it receives focus, then returns to the top-left corner.

The current item is expressed with `aria-current`, and this site uses three values, each with a real meaning:

- In `layouts/_partials/menu.html`, the menu entry that is the current page gets `aria-current="page"`; an ancestor entry that merely contains the current page gets `aria-current="true"` — different meanings, different values.
- `layouts/_partials/sidebar.html` marks the link to the current page with `aria-current="page"` and expands only the branch it lives in.
- `layouts/_partials/breadcrumbs.html` marks the last crumb — an unlinked `<span>` — with `aria-current="page"`.
- `assets/js/modules/toc.js` sets `aria-current="true"` on the heading the reader is looking at and removes it on the way out.

## Machine-readable dates and language attributes

Every date in `layouts/_partials/page-meta.html` is written as `<time datetime="YYYY-MM-DD">`, while the visible text comes from `site.Params.dateFormat` — `2006 年 1 月 2 日` in Chinese. The two are deliberately different: display should match the reader's habit, `datetime` has to be parseable, so changing the display format never touches the data.

Two traps in this layer are already handled. `.Date` is a struct rather than a pointer, so `{{ with .Date }}` is always true; every date is therefore guarded with `.IsZero`, and an undated page never prints `0001-01-01`. And the "updated" entry appears only when `Lastmod` falls on a different day from `Date`, so the same date is not printed twice side by side.

Language attributes live on `<html>` in `baseof.html`: `lang` takes `site.Language.Locale` (`zh-CN` on a Chinese page, `en-US` on an English one) and `dir` takes `site.Language.Direction`, falling back to `ltr` when it is absent. Both values are reused by the hreflang links, by Open Graph's `og:locale` and by the JSON-LD, so they have exactly one source.

## Images, tables and text for assistive technology

Images in body text go through `layouts/_markup/render-image.html`. The `alt` attribute takes the text inside the Markdown brackets, stripped of markup, so writing `![The site's CSS directory](…/tree.png)` means a screen-reader user hears exactly that sentence. When Hugo knows the intrinsic size it adds `width` and `height` — which is what stops the layout jumping as images load — and the element always carries `loading="lazy"` and `decoding="async"`. Giving an image a title (`![alt](img.png "A caption")`) turns it into a `figure` with a `<figcaption>`, which is how the visible caption and the alternative text stay separate concerns.

Tables go through `layouts/_markup/render-table.html` and are always wrapped in `.table-wrap`, so on a narrow screen the table scrolls horizontally instead of widening the page. Block attributes land on the `<table>` element, for example `{#size-table .table--compact}`, which lets you reference it as `#size-table` from elsewhere.

{{< tip >}}
Markdown has no caption element for tables, and there is no reason to fight that: introduce the table with a sentence or a small heading and give it an `id`, and both readers and crawlers can locate it. That is far clearer than a bold table row used as a heading.
{{< /tip >}}

When text is meant for assistive technology only, use `.visually-hidden` or `.sr-only` from `assets/css/base.css`. They move content out of the viewport rather than applying `display: none`, so screen readers still announce it; the selector carries `:not(:focus, :active)`, which means the element becomes visible the moment it is focused — exactly how a control like the skip link is built.

## Print styles and reduced motion

`assets/css/print.css` hides the header, footer, sidebar, table of contents, search box, pager and copy buttons inside `@media print`, collapses the three-column layout to one, and appends external link targets with `::after` — you cannot click on paper, so the destination has to be printed. Elements that should exist only on paper can use `.print-only`.

Motion respects the system setting: the `@media (prefers-reduced-motion: reduce)` block at the top of `base.css` resets `scroll-behavior` to `auto` and compresses animation and transition durations to `0.01ms`. The rule applies to every element, so a new interaction does not have to handle it separately.
