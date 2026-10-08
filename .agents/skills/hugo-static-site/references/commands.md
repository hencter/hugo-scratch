# Commands

## Generate the reference; do not transcribe it

Hugo writes its own CLI documentation, for the version you actually run:

```bash
hugo gen doc --dir <output-dir>   # one Markdown file per command, front matter included
hugo <command> --help             # a single command
```

The documentation's command pages are themselves output of `hugo gen doc`
(<https://gohugo.io/commands/hugo_gen_doc/>), so they describe the version the docs site was
built with — not yours. Per-command public page: `https://gohugo.io/commands/<slug>/`, e.g.
`hugo_server` → <https://gohugo.io/commands/hugo_server/>.

Transcribing a flag table is how a reference goes stale silently, so this file carries only what
the generator does not: which commands the workflow uses, the flags it prescribes, and the
differences worth knowing.

## Commands this skill's workflow uses

| Command | Why |
| --- | --- |
| `hugo version` | decides which config keys, commands, and edition apply |
| `hugo build` | the build; `hugo` alone is the same command |
| `hugo server` | dev server and watcher — never the only verification (G13) |
| `hugo config` | effective settings including defaults (`--format`, `--printZero`) |
| `hugo list all` (+ `drafts`, `future`, `expired`, `published`) | content inventory from Hugo's own page model |
| `hugo new content` | create a page from an archetype |
| `hugo new project` | scaffold a project (older tutorials say `hugo new site`) |
| `hugo gen chromastyles` | stylesheet for the current syntax highlighter |
| `hugo mod init\|get\|tidy\|clean\|vendor\|graph\|verify\|npm pack` | module and Node dependency work |
| `hugo deploy` | sync `public/` to a configured remote |

## Flags this workflow prescribes

Not a reference: `hugo <command> --help` prints the full, authoritative list for your version
(<https://gohugo.io/commands/hugo/>). These are the ones worth reaching for, and what each is for:

| Flag | Effect |
| --- | --- |
| `--ignoreCache` | ignore the configured file caches — reach for it when a result makes no sense |
| `--panicOnWarning` | fail on the first WARNING, so deprecations cannot be scrolled past |
| `--printPathWarnings` | duplicate target paths and similar collisions |
| `--printUnusedTemplates` | templates nothing reaches |
| `--printI18nWarnings` | missing translations |
| `--printMemoryUsage` | memory use at intervals |
| `--templateMetrics`, `--templateMetricsHints` | template execution metrics |
| `--logLevel debug\|info\|warn\|error` | verbosity |
| `--gc`, `--minify`, `--cleanDestinationDir` | post-build housekeeping |
| `-D`, `-E`, `-F` | include drafts / expired / future content |
| `-e/--environment`, `--config`, `--baseURL`, `-d/--destination`, `-s/--source`, `-t/--theme`, `--themesDir` | build inputs and outputs |

Defaults are deliberately absent: `hugo <command> --help` prints them for the installed version.

## What the generator does not tell you

- **Observed:** six parent commands (`hugo new`, `hugo list`, `hugo convert`, `hugo gen`,
  `hugo mod`, `hugo completion`) document no synopsis line, only their subcommands. Do not invent
  one for them.
- **Observed:** `hugo build` and `hugo` carry identical option blocks; only the `--help` text
  differs. `hugo build` is the documented spelling.
- **Documented:** `hugo gen chromastyles --omitEmpty` is marked deprecated on its own page
  (it is no longer needed).
- **Not documented:** nothing else in the command pages is marked deprecated/introduced at a
  version — do not assume a flag carries one.
