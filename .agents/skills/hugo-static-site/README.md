# hugo-static-site

A skill for building, updating, and verifying Hugo static sites — theming and multi-theme
layering, SEO head output and structured data, content in bulk, localized teaching-oriented
documentation — and for diagnosing the build failures that Hugo attributes to the wrong file.

It is written for agents in general, not for one product: the files are plain Markdown with no
scripts, no dependencies and no absolute paths, and the install contract below is the same
whatever loader you use.

It exists because a 200-page Hugo site was built the hard way: the traps in
[`references/gotchas.md`](references/gotchas.md) each cost a real debugging cycle, and one of them
(`HAHAHUGOSHORTCODE`) had silently prevented a page from rendering since the day it was written.

## Contents

```text
hugo-static-site/
├── SKILL.md                        # workflow + iron rules (loaded as the skill)
├── README.md                       # this file
├── INSTALL-PROMPT.txt              # copy-paste install prompt (agent-agnostic; see Install)
└── references/
    ├── commands.md                 # command notes: what the workflow uses (reference: `hugo gen doc`)
    ├── dates.md                    # date fields, time zones, localized formats, relative time
    ├── gotchas.md                  # G1…G26: symptom → cause → fix
    ├── i18n.md                     # optional multilingual setup, switcher, i18n strings
    ├── seo.md                      # head tags, JSON-LD pitfall, sitemap/robots, performance
    ├── shortcodes.md               # authoring custom shortcodes: notation, methods, nesting
    ├── site-structure.md           # theme layers, front matter, navigation, i18n
    ├── teaching-layer.md           # human/machine doc parity: front-matter contract, shared partials
    ├── versioning.md               # what to track, gitInfo, commit-backed "last updated"
    └── versions.md                 # version-keyed renames and defaults
```

Thirteen files in total. A published mirror of this folder, with a machine-readable manifest
(`path`, `bytes`, `sha256`, `url`, `rawUrl` per file), lives at <https://hugozh.cn/skill/> —
`https://hugozh.cn/skill/skill-manifest.json`.

**No scripts, and nothing transcribed that Hugo can generate.** Verification uses Hugo's own
documented commands (`--printPathWarnings`, `--printUnusedTemplates`, `--printI18nWarnings`,
`--panicOnWarning`, `--templateMetrics`, `hugo config`, `hugo list all`) plus two plain `grep`
commands for the two source-level traps no flag reports. The CLI reference, the settings table and
the highlight stylesheet all have generating commands — `hugo gen doc`, `hugo config`,
`hugo gen chromastyles` — so the reference files only say where those live and which flags this
workflow prescribes; what cannot be generated (the trap catalogue, the workflow) is written down.

## Install

**Copy [INSTALL-PROMPT.txt](INSTALL-PROMPT.txt) into your agent and let it install itself.**

The prompt deliberately contains **no product-specific path** — no `.dsh`, no `.claude`, no
`.cursor`. It cannot: every agent's skills/rules directory, loading mechanism and project-vs-user
support differs, so any hard-coded directory silently fails for everyone else. (This skill's own
docs got that wrong twice — first pinning `~/.dsh/skills/`, then treating `.dsh` as the default.)

What the prompt does fix is only what makes the install *checkable*; everything else is left to the
agent, which knows its own convention better than this file does:

| Fixed (otherwise unverifiable) | Left to the agent |
| --- | --- |
| the manifest's `path` and `sha256` for all files | which directory, and what it is called |
| internal relative paths preserved (`references/` never flattened) | which loading/registration mechanism |
| every file hash-verified after writing | project-level or user-level install |
| the agent must state its identity and basis **before** acting | whether a session restart is needed |

The prompt also asks the agent to look for an existing project convention first
(`AGENTS.md`, `CLAUDE.md`, `.cursor/rules`, `.<you>/`) and follow it rather than inventing a
mechanism the project does not have.

**Materials.** The manifest at <https://hugozh.cn/skill/skill-manifest.json> lists each file's
`path`, `bytes`, `sha256` plus two download locations (`url` site mirror, `rawUrl` repository).
`sha256` matches the **published bytes** (UTF-8, no BOM, LF); it proves "identical to what was
published", not "suitable for your project".

**No skills mechanism?** These are thirteen plain Markdown files with no executable code. Read
them into context as reference documentation, or distil the rules into whatever rules file the
agent does support — there is nothing to install. You can also read the site mirror directly:
<https://hugozh.cn/skill/SKILL.md>.

**Fetching without a skills loader.** Either `git clone` the repository and copy the folder, or
fetch each manifest `url` per file. Prefer the manifest route when hashes must match exactly:
`git` rewrites line endings on some platforms (notably Windows), so a clone can hash differently
while the content is identical.

## Use

- Ask for a Hugo task ("add a section to the site", "translate these pages", "the build fails
  with …") and the skill's rules apply once loaded.
- Verify with the commands in `SKILL.md` → *Build and verify*; `references/commands.md` lists the
  commands the workflow uses and the flags it prescribes, and `hugo gen doc --dir <dir>` produces
  the full CLI reference for your version.

## Scope and limits

- The reference files record documentation plus observed behaviour; each claim is labelled
  *documented*, *observed*, or *not documented* (see `SKILL.md` → *Sources and the citation rule*).
- Version claims in `references/versions.md` were observed against a **0.167.0** documentation
  snapshot. Verify against the installed binary (`hugo version`, `hugo config`) before relying on
  them.
- Only Hugo's own exit status proves a build. Everything in this skill is preparation for reading
  that output correctly.
