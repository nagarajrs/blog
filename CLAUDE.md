# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Hugo binary on this machine (until it's on PATH after shell restart):
```
/c/Users/nagar/AppData/Local/Microsoft/WinGet/Packages/Hugo.Hugo.Extended_Microsoft.Winget.Source_8wekyb3d8bbwe/hugo.exe
```

After a new shell session, `hugo` should be available directly.

```bash
# Run dev server (live reload, includes drafts)
hugo server -D

# Build for production
hugo --minify

# New post using the archetype
hugo new content posts/my-post-title.md
```

## Architecture

- **Theme**: PaperMod, installed as a git submodule at `themes/PaperMod/`. Do not edit theme files directly — override via `assets/css/extended/custom.css` or by copying templates into `layouts/`.
- **Config**: All site config in `hugo.toml`. PaperMod-specific params live under `[params]`.
- **Content**: Markdown files in `content/posts/` (blog posts) and `content/` root (standalone pages like About).
- **Custom CSS**: `assets/css/extended/custom.css` — PaperMod automatically merges files in this directory.
- **Deployment**: GitHub Actions at `.github/workflows/deploy.yml` builds on push to `main` and deploys to GitHub Pages.

## Writing Posts

Frontmatter for posts:
```yaml
---
title: "Post Title"
date: 2026-04-03
draft: false          # true = not published
tags: ["go", "systems"]
categories: ["Engineering"]
description: "One-line summary shown in post list."
showToc: true         # show table of contents
---
```

Use `<!--more-->` to set the summary cutoff for the post list.

## Deploying to GitHub Pages

1. Create a GitHub repo (e.g., `nagaraj.github.io` for a user site, or any name for a project site).
2. Update `baseURL` in `hugo.toml` to match: `https://nagaraj.github.io/` or `https://nagaraj.github.io/repo-name/`.
3. Push to `main` — the workflow handles the rest.
4. In repo Settings → Pages → Source, select **GitHub Actions**.

## Overriding Theme Templates

Copy the template from `themes/PaperMod/layouts/` into `layouts/` at the root and edit there. Hugo prefers root `layouts/` over theme layouts.
