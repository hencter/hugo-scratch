+++
title = 'License'
linkTitle = 'License'
description = 'CC BY 4.0 for the prose, MIT for the theme and the code: what you may take and what you must keep.'
date = 2026-01-05
weight = 20
+++

## Two sets of terms, one each

The repository holds two kinds of thing, under different terms:

| Material | Where | Licence |
| --- | --- | --- |
| Prose and illustrations | the text and images under `content/` | CC BY 4.0 |
| Theme, templates, styles and scripts | `themes/hugo-scratch-theme`, `layouts/`, `assets/`, `i18n/` | MIT |
| Code samples inside the prose | the fenced code blocks in the articles | MIT |

The dividing line is whether a thing exists to be read or to be run: articles are
works, code is a tool. A single page contains both, so one page can be governed by
both sets of terms at once.

## The prose: CC BY 4.0

The text and images under `content/` are licensed under
[Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).
Plainly: copy it, translate it, adapt it, print it, use it in training material,
sell it — the **only** requirement is attribution.

No particular format is prescribed; something short is enough:

```text
Source: Hugo Scratch — https://scratch.hugozh.cn/
Licence: CC BY 4.0
```

Add a link to the licence and a note on whether you changed anything, and that is
complete. The Chinese and English versions are not sentence-for-sentence
translations, so when you quote, say which language version you used and give that
page's address.

## The code and the theme: MIT

The theme repository, the `layouts/` and `assets/` directories in this site, and the
code blocks inside the articles are MIT licensed:

```text
MIT License

Copyright (c) 2026 Hencter Lew

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

The same text is in the theme repository's root `LICENSE` file, and `license = "MIT"`
in its `theme.toml` agrees with it. MIT asks exactly one thing: **keep the copyright
notice and this licence text**. It sets no condition that you must publish your
changes — that is copyleft, and MIT is not copyleft.

## Third parties and dependencies

The site itself pulls in no third-party front-end code: no script from a CDN, no
fonts, no analytics. The Hugo binary and the esbuild embedded in it are governed by
their own licences; neither is distributed from this repository, and neither is
affected by the two sets of terms above.

When the prose quotes an outside source, that material still belongs to its author;
the licence on this page covers only what this repository produces.

What the site requests in the browser, and why analytics is off by default, is on
the [Privacy](/legal/privacy/) page — the two pages describe one implementation.
The repository is <https://github.com/hencter/hugo-scratch> and the theme is
<https://github.com/hencter/hugo-scratch-theme>.
