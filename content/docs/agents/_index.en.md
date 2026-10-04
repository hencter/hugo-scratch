+++
title = 'Working on this site through an agent'
linkTitle = 'Agent workflow'
description = 'Where an agent enters, where the facts live, how it proves a change, and the two iron rules of content authoring.'
weight = 80
+++

## Who this chapter is for

It is for people who would rather not hand-write every line and let a coding agent take over this repository instead. There is exactly one prerequisite: an accurate `AGENTS.md` at the repository root. That file is the brief an agent reads first — the directory map, the build command, the paths it must not touch, the content rules — and it is the only file that has to be read before work starts. This chapter expands it; it does not replace it.

## Why a conversation is enough

Because deciding whether a page is worth reading, and what a change will drag along, depends on readable fields rather than on the tone of the prose.

- A page's cost and its payoff live in front matter: `difficulty`, `estimatedTime`, `prerequisites`, `outcomes`. `layouts/_partials/facts.html` renders them as a facts panel, and `/pages.json` carries the same values, so a person and an agent see one answer rather than two;
- Chapter order comes from `weight` in each `_index.md`, and a page's place inside its chapter comes from the page's own `weight`, which makes "insert a page and order it" something you can describe and verify;
- Bilingual content is paired: `name.md` is Simplified Chinese, `name.en.md` is English, same directory and same base name. A request that only changes one of the two should be refused.

The practical effect: a sentence like "add a page about caching to the deployment chapter, after GitHub Pages" already contains enough information to locate the files, without explaining the site structure first.

## Three constraints that are not negotiable

1. **Output is not source.** `public/`, `resources/`, `hugo_stats.json` and `.hugo_build.lock` are output or state. Never edit them by hand.
2. **A change has to be provable.** Get exit code 0 from the repository's strict build, then read the rendered `public/**` file instead of restating the source you just wrote.
3. **Shortcode syntax you want to show must be escaped**, and Hugo's internal placeholder must never be pasted into content. Neither rule is about style: breaking either one fails the whole site build, not just that page.

## What is in this chapter

[Working on this site through an agent](/docs/agents/workflow/) is the operating manual: the entry point, where the facts live, how a change is verified, which directories are off limits, the iron rules of content authoring, and a full request traced from the ask to the proof. It also documents the two supported page switches — the table-of-contents shortcode and the front-matter notice banner — so that nobody writes their own HTML for them.

To find out which directory owns what, read [Directory structure](/docs/start/directory-structure/); to see which features the theme actually implements, read the [feature matrix](/docs/reference/feature-matrix/).
