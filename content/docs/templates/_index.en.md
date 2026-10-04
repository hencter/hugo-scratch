+++
title = 'Templates'
linkTitle = 'Templates'
description = "This theme's template system: baseof defines the contract, page kinds supply only main, and output formats decide what else the same content becomes."
weight = 40
+++

This chapter covers templates, but only the conventions this theme actually uses: `layouts/baseof.html` defines the document contract, and `layouts/page.html`, `layouts/section.html` and their siblings fill in a single `main` block. Output formats then publish the same content a second time as Markdown and JSON. There is no `layouts/_default/` tree and no `single.html`.

## What the two pages cover

- [Templates and lookup order](/docs/templates/templates/) walks from `baseof.html` down to `page.html`, explains what each layer reads, which blocks a site can override besides `main`, and how a partial returns a value instead of printing markup;
- [Output formats](/docs/templates/output-formats/) explains why one home page also exists as `index.md`, `llms.txt`, `search.json` and `pages.json`, and who names those files.

Together they trace a request from content to bytes.

## One directory, three naming conventions

The theme's layout directory uses only three kinds of name:

| Directory | What lives there | Examples |
| --- | --- | --- |
| `layouts/` | Page-kind templates and output-format templates | `page.html`, `home.llms.txt`, `list.md` |
| `layouts/_partials/` | Called fragments, some of which return values | `toc.html`, `layout/flags.html` |
| `layouts/_shortcodes/` | Shortcode implementations | `note.html`, `tabs.html` |
| `layouts/_markup/` | Render hooks | `render-image.html` |

Directories whose names start with an underscore are the convention of Hugo's current template system, and a project's own `layouts/` always wins over a theme's. Overriding one partial does not mean editing the theme submodule: drop a file with the same name into `layouts/_partials/` at the repository root.

## Three rules that bite

{{< warning >}}
Go template's `and` / `or` are functions, and **every argument is evaluated**. `and $p $p.Title` still panics when `$p` is nil, because `$p.Title` was already evaluated. To short-circuit, use nested `with`, or compute a safe string first.
{{< /warning >}}

- **`default true` cannot express an explicit `false`**: `default` only takes over when the value is empty, so a caller who writes `false` looks like a caller who wrote nothing, and the value becomes `true`. Switches belong behind `isset`, or behind a direct comparison — `layouts/_partials/layout/flags.html` reads `showSidebar` with `index` followed by an `eq ... nil` test for exactly this reason.
- **`.` is the current context and `$` is the context the template started with.** Once you are inside a nested `range` in a partial and need the page again, write `$` rather than guessing.

## Who reads these templates

A template is not documentation; every region of the page maps to a file. The sidebar comes from `layouts/_partials/sidebar.html`, the on-this-page navigation from `layouts/_partials/toc.html`, and the llms.txt / pages.json links in the footer from `layouts/_partials/footer.html` reading `.OutputFormats`. Before changing a template, find that region in one of those files — it is far faster than working backwards from a CSS class name.
