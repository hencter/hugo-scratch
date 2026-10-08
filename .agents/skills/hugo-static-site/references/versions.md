# Version-keyed changes worth knowing

Observed against a Hugo **0.167.0** documentation snapshot. Version numbers are the ones the
upstream docs attach to each change; confirm for the version actually installed, because several
of these moved defaults as well as names.

| Version | Change that bites |
| --- | --- |
| 0.140.0 | `details` shortcode; `js.Build`: `loaders`, `platform`, `sourcesContent`; `disableDefaultLanguageRedirect` |
| 0.144.0 | permalink token `:contentbasename`; `js.Build` `drop` |
| 0.146.0 | new template system: `layouts/_partials/`, `layouts/_shortcodes/`, `layouts/_markup/`, kind templates (`home.html`, `page.html`, `section.html`, `all.html`); `templates.Current`. Legacy `layouts/_default/…` still resolves but is not the current layout |
| 0.149.0 | permalink tokens `:sectionslug`, `:sectionslugs` |
| 0.153.0 | `cascade` page matcher: `lang` deprecated, `sites` matrix added; content roles (`[roles]`, `defaultContentRole`) |
| 0.155.0 | `.Exif` deprecated (`.Meta` from 0.155.3); imaging `quality`/`hint`/`compression` deprecations begin; metadata is not carried through transformations |
| 0.158.0 | `languageCode` → **`locale`**; language `Lang`/`LanguageCode`/`LanguageDirection`/`LanguageName` deprecated (use `.Locale`, `.Name`); `hugo new site` → **`hugo new project`**; multilingual `languageName` → `label`; content security `allowContent` defaults hardened |
| 0.159.2 | the **deploy** edition appears in the installation matrix (standard / deploy / extended / extended+deploy) |
| 0.161.0 | security `node.permissions` additions |
| 0.162.0 | security `allowContent`; omitting a config category name; AVIF imaging |
| 0.163.0 | imaging top-level `quality`/`compression`/`hint` deprecated in favour of per-format `[imaging.avif|jpeg|webp|…]` |
| 0.164.0 | `hugo gen chromastyles --mode` / `--modeSelector` |
| 0.165.0 | `js.Build` `importContext`; `.Data.Artifacts` on built JS resources with `<link rel="preload">` |
| 0.166.0 | security `node.permissions` refinements |
| 0.167.0 | `slug` inheritance from section/taxonomy/term pages to their descendants; `cleanDestinationDir` key set replaced; build output prints the site matrix (`[zh-cn v1.0.0 guest]`) |

## How to use this table

1. Run `hugo version` first and read the table only for versions **at or below** it.
2. When a key or command name is in doubt, ask the installed binary rather than a tutorial:
   `hugo config`, `hugo env`, `hugo <command> --help`.
3. Deprecations are announced before removal (INFO for three minor versions, WARN for more, then
   an error). Warnings printed during a build are part of the result, not noise.

## Edition check

The installation matrix distinguishes **standard**, **deploy**, **extended** and
**extended+deploy** (0.159.2+). `hugo version` prints `extended` when present. Dart Sass
transpiling, WebP encoding and `hugo deploy` depend on the edition; a build that fails on a
pipeline step may simply be running the wrong edition.
