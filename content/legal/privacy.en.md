+++
title = 'Privacy'
linkTitle = 'Privacy'
description = 'What this site requests in the browser, what it deliberately does not, and when a third-party request can happen at all.'
date = 2026-01-05
weight = 10
+++

## The short version

**A page requests same-origin resources only.** The stylesheet, the script, the
images and `search.json` all come from this site; nothing outside
`https://hencter.github.io/hugo-scratch/` is contacted. In its default
configuration the site sets no cookies and reports no visit to any analytics
service. Each claim is spelled out below, with a way to check it yourself.

## What a page actually requests

| Resource | Origin | Notes |
| --- | --- | --- |
| `/css/bundle.min.<hash>.css` | this site | the theme's `main.css` plus the site's `custom.css`, concatenated |
| `/js/main.<hash>.js` | this site | the single bundle `js.Build` produces |
| the images on the page | this site | resolved from the page bundle, `assets/` or `static/` |
| `/search.json` | this site | fetched **only when the reader opens the search dialog** |
| icons and the webmanifest | this site | the favicons and site manifest under `static/` |

Both built assets carry a content hash in the filename and an `integrity`
attribute: when the hash does not match, the browser refuses to execute the file.
That is an **integrity** guarantee, not a tracking mechanism.

## Analytics is off by default

`layouts/_partials/analytics.html` checks two things: whether `[params.analytics]`
holds a value, and whether this particular build is a production build. Both have to
be true before any tag is emitted.

The repository's values are empty right now:

```toml
[analytics]
  googleAnalytics = ''
  plausibleDomain = ''
```

So the site as it stands has no analytics script at all and nothing that would
contact Google or Plausible. The configuration also sets
`disable = true` under `[privacy.googleAnalytics]`.

Filling those values in changes the picture: Google Analytics' gtag loads from
`googletagmanager.com` and Plausible loads from `plausible.io`, and **only in a
production build**. A visit during `hugo server` never enters the statistics — the
price being that a local preview shows you no data either.

## Three common third-party requests that are simply absent

**No font CDN.** The font stack is a system stack, written out in
`assets/css/tokens.css` as `--font-sans` and `--font-mono` (`-apple-system`,
`Segoe UI`, `ui-monospace` and so on). Fonts therefore cause no network request at
all, and the page never flashes while waiting for one.

**No YouTube embed.** The theme overrides Hugo's built-in `youtube` shortcode to
render a **link** to `youtube-nocookie.com` rather than an `<iframe>` player. The
built-in version contacts YouTube and writes cookies as soon as the page opens;
here the request waits until the reader clicks through — and by then the reader has
left the site anyway.

**No comment widget.** `layouts/_partials/comments.html` renders a link to an issue
in the repository. It points at `github.com`, and a link issues no request until it
is clicked.

## The search index is fetched on demand

The search dialog is backed by a JSON index: `/search.json` (or `/en/search.json` in
English). It is generated at build time by the `search` output format on the home
page, but it does **not** load with the page.

`load(params)` in `assets/js/modules/search.js` calls `fetch` only when the reader
first opens the search dialog, and it passes `credentials: 'same-origin'`. Once
fetched, the index stays in memory, so further searches in the same visit make no
second request. A reader who never searches never downloads a byte of it.

## How to verify all of this

1. Open any page, press {{< kbd "F12" >}} for DevTools, switch to the Network panel
   and reload. Every request in the list should be to this site.
2. Open the search dialog ({{< kbd "Ctrl" >}} + {{< kbd "K" >}}). Only now does a
   `search.json` request appear.
3. Look under Application / Storage: the site writes no cookies. Local state such as
   the colour-theme choice stays in this browser.
4. Repeat in a clean private window. The request list should be identical — no extra
   third-party host, and no cached script smoothing over the difference.

If you see another domain on your own machine, it almost certainly came from a
browser extension or a proxy rather than from this site.
