+++
title = 'Two languages for one site'
description = 'How the bilingual build is wired: .en.md pairing, the language switcher fallback, hreflang versus og:locale, and per-language menus and date formats.'
date = 2026-03-24
authors = ['Hencter Lew']
tags = ['Hugo', 'multilingual']
categories = ['engineering']
series = ['Building a Hugo site from scratch']
+++

## The pairing rule: `.en.md`, not two directory trees

This site maintains **one content tree**. Simplified Chinese is the default
language, so it owns the plain filename; English is the same base name in the same
directory with an `.en` suffix:

```text
content/
├── _index.md / _index.en.md                     # / and /en/
├── about/index.md / about/index.en.md           # /about/ and /en/about/
└── blog/hello/index.md / blog/hello/index.en.md # /blog/hello/ and /en/blog/hello/
```

Hugo pairs files by "same directory, same base name, different language suffix" —
the directory name plays no part. Pairing worked when `.Translations` and
`.AllTranslations` are non-empty, which is exactly what `lang-switcher.html` reads.

## The other approach, and why the two must not be mixed

The alternative is one directory per language: `content/zh-cn/**` and
`content/en/**`, each tree with its own `contentDir`. That suits a large project whose
two sides are maintained by different people and drift out of step for months. Mixing
the two does not work. Put `content/zh-cn/legal/privacy.md` and
`content/legal/privacy.en.md` in one repository and Hugo pairs neither, because it
only ever sees one file per language: the `.en.md` suffix is a pairing signal only
while **both languages share the same tree**. Every page then has an
`.AllTranslations` of length 1, the site is silently monolingual, and nothing warns
you.

## Why the language switcher falls back to the home page

Language controls are where 404s come from, so `lang-switcher.html` never builds a
URL by hand:

```go-html-template
{{- range hugo.Sites -}}
  {{- $url := .Home.RelPermalink -}}
  {{- with $page.Translations -}}
    {{- range . -}}
      {{- if eq .Language.Lang $lang }}{{ $url = .RelPermalink }}{{ end -}}
    {{- end -}}
  {{- end -}}
{{- end -}}
```

The rule is "prefer this page's translation, fall back to that language's home page":
`$url` starts as the other language's home page and is overwritten only when a
translated page of the same language exists. `hugo.Sites` includes the current
language, so one control renders identically on both sides. The trade-off is that on
an untranslated article the switch lands on the home page, and nothing says so.

## `hreflang` versus `og:locale`: hyphens against underscores

One language tag produces two labels in two formats. `hreflang` is BCP-47 and uses
**hyphens** (`zh-CN`); `og:locale` is Open Graph's dialect and uses **underscores**
(`zh_CN`). Both derive from the same site locale, so one of them has to be rewritten:

```go-html-template
<meta property="og:locale" content="{{ replace site.Language.Locale "-" "_" }}">
```

`layouts/_partials/head/alternates.html` emits `Locale` unchanged and adds an
`x-default` entry. `x-default` answers "who gets this when no language fits better",
and it explicitly does **not** follow language order: add a language, change a
`weight`, and the order moves. So it is read from `params.seo.xDefaultLang`
(`zh-cn` here).

## Menus and date formats are written per language

Menus do not go through the i18n catalogue. The Chinese and English menus live in
`config/_default/menus.zh-cn.toml` and `menus.en.toml`, each declaring its own
`[[main]]` entries and associating them with a page through `pageRef = '/docs'`. A
menu label is navigation structure rather than a string inside a template, a missing
i18n key would warn on every page, and an English site usually replaces or reorders
entries instead of translating them one for one.

Date formats are given per language too, under `[<lang>.params]` in
`config/_default/languages.toml`:

```toml
[zh-cn.params]
  dateFormat = '2006 年 1 月 2 日'

[en.params]
  dateFormat = 'January 2, 2006'
```

The `:date_long` localisation token is not usable here: its localisation data does not
cover every language, and zh-CN falls back to English, which puts English dates on a
Chinese page.

## How interface strings reach the JavaScript

Strings used inside templates go through Hugo's i18n: `i18n/zh-cn.toml` and
`i18n/en.toml` are key-for-key equivalents, templates call
`{{ T "searchNoResults" }}`, and `--printI18nWarnings` complains when one side is
missing a key.

But the search dialog and the copy buttons write their labels from JavaScript at
runtime, where no template can reach them. Those go through `js.Build`'s `params`:
`head/js.html` assembles the `T` results into an `i18n` dictionary, and esbuild
exposes it as the virtual module `@params`. Each language therefore gets its own bundle
and its own hash.
