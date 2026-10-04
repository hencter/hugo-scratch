+++
title = 'Publishing to GitHub Pages'
linkTitle = 'GitHub Pages'
description = 'Build and publish this project site with GitHub Actions: submodule checkout, a pinned Hugo version, the strict build, and the Pages source switched to Actions.'
date = 2026-03-02
weight = 10
difficulty = 'intermediate'
estimatedTime = 25
prerequisites = ['/docs/start/quick-start/', '/docs/configuration/site-config/']
outcomes = ['Explain why baseURL must match the address the site is actually served from', 'Write a publishing workflow with submodule checkout and the strict build', 'Switch the repository Pages source to GitHub Actions', 'Know which DNS records and which configuration key a custom domain needs']
tags = ['deployment']
+++

## Why baseURL has to match the address the site is served from

`baseURL` in `config/_default/hugo.toml` is `https://scratch.hugozh.cn/`. This site is published on a custom domain and served from the **root**, so no generated URL carries a subpath. `defaultContentLanguageInSubdir = false` puts Simplified Chinese at `/` and English at `/en/`, so both languages live under that same root.[^1]

Hugo builds every absolute URL from `baseURL`: the `<link>` and `<script>` tags for the stylesheet and the scripts, each page's canonical URL, the Open Graph tags, `sitemap.xml` and the feeds. Change it to some other address and the pages still open, but every asset now points there — the result is content with no styling.

{{< warning >}}
Getting it wrong shows up as 404s for assets, not as a build error. The same is true in the other direction: the default URL of a GitHub Pages *project* site looks like `https://<owner>.github.io/<repo>/`, and publishing there means `baseURL` must carry `/<repo>/`; publish on a custom domain while keeping a subpath and every asset points at a location that does not exist.
{{< /warning >}}

## Repository settings

In the repository, open **Settings → Pages** and set **Source** to `GitHub Actions`. The change is immediate; there is no Save button.

- While the source stays on `Deploy from a branch`, GitHub runs Jekyll itself and your workflow is never used as the publishing source;
- This source needs no `gh-pages` branch and no committed `public/` — the workflow uploads the artifact;
- When you publish from a custom GitHub Actions workflow, GitHub does not create a `CNAME` file, and an existing one is ignored. Put the custom domain in the Pages settings rather than a file under `static/`;
- Tick **Enforce HTTPS** after the first successful deployment; issuing the certificate takes a while.

## A workflow that works

`.github/workflows/pages.yml`:

```yaml
name: Build and deploy

env:
  HUGO_VERSION: 0.167.0

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v6

      - name: Install Hugo
        run: |
          curl -sfL --output-dir "${{ runner.temp }}" -O \
            "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir -p "$HOME/.local/hugo"
          tar -C "$HOME/.local/hugo" -xf "${{ runner.temp }}/hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "$HOME/.local/hugo" >> "$GITHUB_PATH"

      - name: Build
        run: |
          hugo build \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --ignoreCache \
            --panicOnWarning \
            --printPathWarnings \
            --printUnusedTemplates \
            --printI18nWarnings

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v5
        with:
          path: ./public

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

What each part is for:

- **Submodules**: `submodules: recursive` checks out `themes/hugo-scratch-theme` as well. Without it the build reports `found no layout file for "html" for kind "page"`, which never points at the real cause; locally the fix is `git submodule update --init --recursive`.
- **Full history**: the site sets `enableGitInfo = true` and `[frontmatter] lastmod = [':git', 'lastmod', 'date']`, so a shallow clone pushes "last updated" back to the front-matter value.
- **Hugo version**: the step downloads the **non-extended** `hugo_<version>_linux-amd64.tar.gz`. The theme declares `[module.hugoVersion] extended = false` in `themes/hugo-scratch-theme/hugo.toml`, and this repository has been built on the standard 0.167.0 binary — the Tailwind integration itself does not need the extended edition. `min = '0.146.0'` is the floor: whatever `env.HUGO_VERSION` pins must be at least that, and here it is 0.167.0.
- **Build command**: `hugo build` is the same as a bare `hugo`, and the flags are exactly the strict build the repository uses locally. `--panicOnWarning` fails on the first WARNING, so a release never ships with, say, an internal link that resolves to no page.
- **Publishing**: `actions/upload-pages-artifact@v5` packages `public/` as the Pages artifact and `actions/deploy-pages@v5` publishes it. `permissions` needs at least `pages: write` and `id-token: write`, and `environment: github-pages` is what puts the live URL on the Actions job page.
- **One install step that cannot be skipped**: `npm ci`. The Tailwind stage of the stylesheet runs the Tailwind v4 CLI installed at the site root; it is not part of Hugo, so CI has to install dependencies before the verification build. `actions/setup-node` plus `npm ci` is enough, and the lockfile makes the result repeatable. Miss it and the failure is a missing `tailwindcss` executable rather than a broken page.
- **What is missing on purpose**: Dart Sass and Go are not installed. Once dependencies are installed the build is fully offline, so the conditional toolchain steps in the upstream example are dead weight here, and `config/production/hugo.toml` deliberately leaves `[minify]` off so the built HTML stays readable.

{{< note >}}
`--baseURL "${{ steps.pages.outputs.base_url }}/"` is the address reported by `actions/configure-pages`, which brings both the trailing slash and the subpath with it. If you would rather have a single source of truth, delete the line — the committed `baseURL` expresses the same address.
{{< /note >}}

## Publishing from a branch other than main

The **file name does not matter**; what selects a workflow is its own `on:` trigger. When the branch you publish from is not the default branch:

- Add it to `on.push.branches` (for example `branches: [main, release]`), or replace the list with `branches: [release]`;
- Keep `workflow_dispatch:` so the workflow can be run by hand from the Actions tab; `actions/checkout` then checks out the ref that triggered it;
- To run two publishing lines at once, use two workflow files (for example `pages.yml` and `pages-release.yml`), each with its own `branches` list; or keep one file and guard the `deploy` job with `if: github.ref == 'refs/heads/main'`;
- Give both publishing workflows the same `concurrency.group` (the example uses `pages`), or they can write to Pages at the same time and the last one to finish wins.

## Publishing somewhere else: Netlify and Cloudflare Pages

Both hosts follow the same model: connect the repository, give a build command and an output directory. Four things have to line up for this repository — the build command (the strict one above, or a bare `hugo`), the output directory `public/`, the Hugo version pinned through the `HUGO_VERSION` environment variable (again at least 0.146.0, non-extended is fine), and submodule initialisation during the platform's clone. That last one is the usual trap: when the platform does not run `git submodule update --init --recursive`, the symptom is identical to a local checkout that skipped the submodule.

The one thing that has to line up when you move hosts is still `baseURL`: change it to their domain (usually `https://<project>.netlify.app/` or `https://<project>.pages.dev/`, both of which serve from the root). The `/<repo>/` prefix belongs to the default URL of a GitHub Pages project site only; hand a prefixed address to Netlify or Cloudflare Pages and every asset points at a subpath that does not exist.

## Custom domain

This site is published exactly this way: the custom domain `scratch.hugozh.cn` serves it from the root. Four steps:

1. Enter the domain under **Settings → Pages → Custom domain** and save. As noted above, an Actions deployment creates no `CNAME` file and needs none.
2. Configure DNS. For an apex domain (`example.com`) add four A records: `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`. For `www`, add a CNAME record pointing at `hencter.github.io` — without the repository name. This site uses a subdomain, so one CNAME record pointing at `hencter.github.io` is enough.
3. Change `baseURL` in `config/_default/hugo.toml` to the custom domain, ending in `/`: `baseURL = 'https://scratch.hugozh.cn/'`. A custom domain is served from the root, so the address must not carry a `/<repo>/` subpath, or you get asset 404s. Changing the domain later touches this one value and nothing else.
4. After DNS has propagated, tick **Enforce HTTPS** in the Pages settings.

Once the domain is set up, run the strict build again and read `public/sitemap.xml` plus one page's `public/**/index.html` to confirm that canonical and `og:url` now carry the new domain. That catches the problem earlier than a browser refresh does.

[^1]: Upstream documentation: [Host on GitHub Pages](https://gohugo.io/host-and-deploy/host-on-github-pages/)
