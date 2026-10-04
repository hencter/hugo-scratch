+++
title = 'Legal and licensing'
linkTitle = 'Legal'
description = 'Privacy, licensing and terms of use.'
weight = 90
+++

## Why this section stands on its own

Privacy and licensing are neither a tutorial nor a build note: they have a different
audience and a much slower rate of change. So they are de-prioritised separately in
`config/_default/hugo.toml` — a `[[cascade]]` scoped to `/legal/**` sets the sitemap
`changefreq` to `yearly` and `priority` to `0.2` — and they sit outside the main
navigation.

Each page here describes an **actual implementation** rather than boilerplate:

- [Privacy](/legal/privacy/): which same-origin resources the browser requests, the
  one condition under which a third-party request happens, and why analytics is off
  by default.
- [License](/legal/license/): CC BY 4.0 for the prose, MIT for the theme and the
  code.

## When these pages have to change

Change any of the following and the matching page changes in the same commit — their
correctness is coupled:

| Change | Page to update with it |
| --- | --- |
| put a value in `[params.analytics]` | [Privacy](/legal/privacy/) |
| a shortcode starts making a third-party request | [Privacy](/legal/privacy/) |
| `params.images` or the font stack changes | [Privacy](/legal/privacy/) |
| the licence terms of content or code change | [License](/legal/license/) |

The reason is plain enough: a privacy notice that cannot say what the site requests
is worth nothing, and a site that prints "no analytics" in its prose while turning
analytics on in its configuration has failed at both jobs.

## Not legal advice

These pages state **what this site actually does**: what it loads, what it does not,
and how it is licensed. They are not legal advice and they make no claim to fit your
jurisdiction. If you turn the scaffold into a site of your own, treat them as a
starting point and rewrite them — especially once you add analytics, comments, a font
service, or any other third-party resource.
