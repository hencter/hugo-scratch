+++
title = 'Start'
linkTitle = 'Start'
description = 'Get the repository onto your machine, run it, and learn which directories are safe to edit.'
date = 2026-01-10
weight = 10
difficulty = 'beginner'
estimatedTime = 15
prerequisites = ['/docs/']
outcomes = ['Run this site locally', 'Know which directories are build output']
tags = ['Hugo']
+++

This section has two pages, and they are the precondition for everything after them: get the site running, then work out which directories you are looking at are source and which are output. Skipping this step and editing a template straight away usually means editing a file under `public/`, which the next build overwrites.

## The two pages here

[Quick start](/docs/start/quick-start/) gets the site onto `localhost`. It begins with the Hugo version requirement and then handles the step this repository gets wrong most often: the theme is attached as a git submodule, so the clone command needs `--recurse-submodules`. A plain `git clone` leaves `themes/hugo-scratch-theme` empty; Hugo then refuses to render, and all it says is that it found no layout file — never that a submodule is missing.

[Directory structure](/docs/start/directory-structure/) prints the whole repository tree and explains each directory in turn. It includes the list of directories to leave alone: `public/`, `resources/`, `.hugo_build.lock` and `hugo_stats.json` are output or state files. Hand-editing them either gets overwritten or leaves the next build reading a state that contradicts itself.

## What you need

A Hugo binary at 0.146 or newer — the floor the theme declares under `[module.hugoVersion]` — a copy of the repository with submodules, a terminal, and one `npm ci` (the Tailwind v4 CLI lives at the site root and only the stylesheet stage uses it). No Sass compiler is needed for the design system or the scripts: Hugo's `css.Build` and `js.Build` handle them entirely, and the fonts and images are plain static files.

Hugo reports its own version with `hugo version`, so run it first. If it prints anything below 0.146, upgrade before going further. Below the floor the build fails while reading the module configuration, not on some page, so it looks as if the site itself is broken.

## What to verify once it runs

After `hugo server` starts, the home page should render with its title, navigation bar and language switcher. A page of unstyled plain text usually means the theme is not loaded rather than that the CSS is missing. Then open [Quick start](/docs/start/quick-start/) itself, change a sentence and save before pressing <kbd>Ctrl</kbd> + <kbd>C</kbd>, and watch whether the browser reloads on its own. Live reload is the main return on a development server; without it you restart by hand after every edit.

## Where to go next

Once the site runs, resist the urge to edit a template. Read [Configuration](/docs/configuration/) first: it explains why this site does not keep a `hugo.toml` at the repository root, and how the theme, `_default` and environment layers merge. With that layer understood, any value you see on any page can be traced back to the file that set it. Without it, you spend your time guessing where a parameter came from.
