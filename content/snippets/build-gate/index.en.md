+++
title = 'The strict build gate'
description = 'The one command that has to pass before any change counts as done.'
# headless = true makes this a headless bundle: Hugo renders its content but
# publishes nothing — no URL, no listing entry, no Markdown twin. It exists only
# to be pulled in by {{< include >}}. Remove the line and the fragment becomes an
# ordinary page at /en/snippets/build-gate/, which is how you can prove the
# difference in one build.
headless = true
+++

Run this from the repository root before claiming any change is done. **Only exit code 0 counts:**

```bash
hugo --ignoreCache --panicOnWarning --printPathWarnings --printUnusedTemplates --printI18nWarnings
```

It blocks five classes of problem at once: deprecated config keys or template methods (`--panicOnWarning`), two pages writing to the same output path, templates nothing reaches, missing translations, and the broken links and images the render hooks report.

The build output is the evidence: a page rendered only if its directory exists under `public/`. A silent console does not mean a page was generated.
