+++
title = 'Structured data'
linkTitle = 'Structured data'
description = 'What each of the four nodes in one JSON-LD @graph contributes, why the string-literal trap makes every consumer reject the output, how the breadcrumb stays in step with the visible trail, and how to validate.'
date = 2026-02-25
weight = 20
difficulty = 'advanced'
estimatedTime = 18
prerequisites = ['/docs/', '/docs/seo/', '/docs/seo/semantic-html/']
outcomes = ['Say what each node of schema.html contributes', 'Explain the jsonify string trap and the safeJS fix', 'Validate one page with the two official tools']
tags = ['SEO']
+++

Structured data is the one part of this chapter you can neither see nor guess: nothing changes on the page, but a block of JSON appears in the output. It is produced by a single file, `layouts/_partials/head/schema.html`, which runs to a hundred-odd lines. Only four decisions in it really need understanding.

## One @graph instead of ten scripts

The template emits exactly one `<script type="application/ld+json">` containing a `@graph` array. A graph rather than several scripts, because of how the nodes reference each other: Organization has its own `@id` (the home URL plus `#organization`), WebSite points at that `@id` through `publisher`, and the page node points at the WebSite `@id` through `isPartOf`. A consumer with one JSON payload can reconstruct publisher, site and page without a second fetch.

`@id` values are built from absolute URLs because they have to stay stable across pages: the same Organization node appears on every page of the site, which is how a search engine knows these are one entity rather than thousands of identically named organisations. The site title and `params.seo.organization` decide `name`, and `params.seo.logo` is resolved through the same image helper Open Graph uses and written as an `ImageObject`.

## What the four nodes are

**Organization** describes the publisher: `@id`, `name`, `url` and, optionally, a logo. When `organization` is unset the value falls back to `site.Title`, so this node never has missing fields.

**WebSite** describes the site itself: `@id`, `url`, `name`, `inLanguage` (from `site.Language.Locale`) and `publisher`. When the home page has the `search` output format it also gains a `potentialAction` — a `SearchAction` whose `target.urlTemplate` points at `search?q={search_term_string}` with a `query-input` naming the parameter. That part is conditional: remove `search` from `[outputs]` and the search action disappears too, rather than leaving a declaration pointing at an empty page.

**The page node** is the main subject, and its type follows the page kind:

- home → `WebSite`;
- other branch pages (sections, taxonomies, terms) → `CollectionPage`;
- regular pages → `WebPage`;
- pages under `blog` with a date → `BlogPosting`.

The remaining fields come from the page: `url` and `name`, `isPartOf`, `inLanguage`, `description` (run through `plainify` to strip markup), `datePublished` and `dateModified` in RFC 3339 form, `wordCount`, `timeRequired`, author and image. Two details are worth spelling out. `timeRequired` is written as `PT<n>M` rather than a bare number and is floored at one minute, because `ReadingTime` is an int and letting it into `math.Max` would make it a float, which then prints as garbage. And `keywords` reads the `keywords` front matter field, not `tags` — tags have their own taxonomy pages, while keywords belong to this one page.

**BreadcrumbList** is the fourth node, generated last because it depends on the other three. Every entry is a `ListItem`: positions start at 1, names and links come from `.Ancestors.Reverse`, and the page itself is appended at the end.

## The string-literal trap and the fix

The single line of this template most worth remembering is the last one. The correct approach is to hand an **object** to `jsonify` and insert the result as JavaScript:

```text
{{ $json := dict "@context" "https://schema.org" "@graph" $graph | jsonify }}
<script type="application/ld+json">{{ $json | safeJS }}</script>
```

Do it the other way round — re-`jsonify` an already rendered string, for instance by piping the return value of a partial straight into `jsonify` — and what you get is a JSON **string literal**: the whole `@graph` is wrapped in quotes with its inner quotes escaped, every consumer sees a string and nothing else, and the verdict is "invalid structured data". This is not a Hugo quirk; `jsonify` encodes a value as JSON, and the JSON encoding of a string is a quoted string.

The second trap is escaping. Go's JSON encoder writes `<`, `>` and `&` as `\u003c`, `\u003e` and `\u0026`, so even a `</script>` inside the payload cannot close the script element early. On top of that, the template replaces `</` with `<\/`, so the behaviour does not regress if a future Hugo drops that escaping. Two layers of protection mean the payload is safe inside HTML and readable by any standard parser.

One precondition makes it safe for a head partial to read body-level information: `layouts/_partials/layout/flags.html` touches `$page.Fragments`, and `baseof.html` calls it **before** rendering `head.html`. So `.WordCount`, `.ReadingTime` and the heading structure are already available while `<head>` is being written, which is where `wordCount` and `timeRequired` get their values.

## BreadcrumbList must match the visible trail

`layouts/_partials/breadcrumbs.html` renders an `<ol>` on the page and `schema.html` produces the matching `BreadcrumbList`; both read the same `.Ancestors.Reverse`. That is a requirement, not a coincidence: a visible trail that disagrees with the structured one counts as a structured-data error, and a search engine treats it as navigation that contradicts the declaration.

So never change one side alone. Add a level to the visible path — a category link before the page title, say — and the JSON-LD has to show it too. Conversely, if a page should not appear in the breadcrumb at all, the fix is to keep it out of `.Ancestors`, not to skip it by hand inside the JSON-LD.

## How to validate

Read the raw output first, then hand it to the official tools. Keep that order, because a tool error usually means the tool never saw the data rather than that the data is wrong:

```bash
hugo --ignoreCache
grep -o '"@type":"[^"]*"' public/docs/seo/structured-data/index.html
```

You should see `Organization`, `WebSite`, `WebPage` (or `CollectionPage` on a branch page) and `BreadcrumbList` once each. If none of them appear, `schema.html` is not being included — check the call order in `layouts/_partials/head.html`, where it is the last head partial.

{{% steps %}}
{{% step "Read the local output" %}}
After a build, open any page's HTML under `public/`, search for `application/ld+json` and paste the JSON into an editor that reports syntax errors. Anything malformed shows up here, before a tool has to declare it invalid.
{{% /step %}}
{{% step "The schema.org validator" %}}
Paste the page URL or the JSON itself into [validator.schema.org](https://validator.schema.org/). It checks vocabulary and field types, so it will tell you when a property does not belong to the type it sits on.
{{% /step %}}
{{% step "Google's Rich Results Test" %}}
Then use the [Rich Results Test](https://search.google.com/test/rich-results) to see what Google actually extracts. It only cares about the rich-result types it supports, so "no rich results found" does not mean the data is wrong; keep the two judgements separate.
{{% /step %}}
{{% /steps %}}

{{< warning >}}
The validation tools have to be able to reach the page. A local `hugo server` address is usually not crawlable, and a preview deployment carries `noindex, nofollow` — which does not stop JSON-LD from being parsed but does stop the tools from seeing the content. The path of least resistance is to validate on a temporary public deployment, or to paste the JSON straight into the schema.org validator.
{{< /warning >}}
