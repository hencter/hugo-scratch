+++
title = 'Deployment'
linkTitle = 'Deployment'
description = 'Hand public/ to a static host and keep baseURL identical to the address the site is actually served from.'
weight = 70
+++

## What this chapter is about

`hugo` produces exactly one directory: `public/`. Deployment is therefore not a vague "put the site online" task but two concrete ones — hand `public/` to a static host, and make sure the `baseURL` used at build time is the address the site is actually served from. The host is interchangeable; `baseURL` has exactly one correct value.

The hard part in this repository is not the platform but whether `baseURL` and the real address line up. `baseURL` in `config/_default/hugo.toml` is `https://scratch.hugozh.cn/`: the site is published on a custom domain and served from the **root**, so no generated absolute URL carries a subpath. Write it as an address that does carry one — for example the default URL of a GitHub Pages *project* site, `https://<owner>.github.io/<repo>/` — and the pages still open, but the stylesheet, the scripts and every internal link return 404: the symptom is content with no styling, not a build error.

The second prerequisite that is easy to miss is the theme: `themes/hugo-scratch-theme` is a git submodule pointing at `https://github.com/hencter/hugo-scratch-theme`. Every build environment has to check out submodules recursively, or Hugo finds no template at all and reports `found no layout file for "html" for kind "page"` — it never says that a submodule is missing.

## Check the build before you ship it

The repository verifies a build with one command, and CI uses the same one:

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

`--panicOnWarning` turns the first WARNING into a failure, so this command is both a build and a check: duplicate output paths, templates nothing references, missing i18n keys and internal links that resolve to no page all surface here. The price is that it is too strict for local experimentation — drop `--panicOnWarning` and only warnings remain.

{{< note >}}
`--printPathWarnings` reports two pages writing the same output path. That never fails the build; it lets the later page overwrite the earlier one, which on the live site looks like a page whose content is wrong.
{{< /note >}}

## What is in this chapter

[Publishing to GitHub Pages](/docs/deploy/github-pages/) is the main line: from the Pages source in the repository settings, through a workflow that checks out submodules, pins a Hugo version, runs the strict build and deploys with `actions/upload-pages-artifact` and `actions/deploy-pages`, to the parts to change when you publish from a branch other than `main`. The same page covers the equivalent setup on Netlify and Cloudflare Pages, plus the DNS records and the `baseURL` change a custom domain requires.

## What this chapter does not cover

- The asset pipeline (CSS and JS bundled by Hugo Pipes) is in the [assets](/docs/assets/) chapter;
- The machine-readable output (`/llms.txt`, `/pages.json`, `/search.json`) is under [output formats](/docs/templates/output-formats/);
- Production-only switches live in `config/production/hugo.toml`; see [site configuration](/docs/configuration/site-config/).
