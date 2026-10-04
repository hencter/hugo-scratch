+++
title = 'Quick start'
linkTitle = 'Quick start'
description = 'Clone, install, start the dev server — three steps to a rendered page.'
date = 2026-01-10
weight = 10
difficulty = 'beginner'
estimatedTime = 10
prerequisites = ['/docs']
outcomes = ['Run this site locally', 'Know which file changes which page']
tags = ['getting-started']
+++

## Clone and start

```bash
git clone --recurse-submodules https://github.com/hencter/hugo-scratch.git
cd hugo-scratch
hugo server
```

## Check that you edited the right place

Open `content/docs/quick-start.md`, change a sentence, save; the browser reloads.
