# Nagaraj's Blog — Setup & Publishing Guide

A technical blog about HPC, AI/ML infrastructure, and life sciences computing at scale.
Built with [Hugo](https://gohugo.io) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod), deployed to GitHub Pages via GitHub Actions.

**Live site**: https://nagarajrs.github.io/blog/

---

## Table of Contents

1. [Directory Structure](#1-directory-structure)
2. [Local Development](#2-local-development)
3. [Writing a New Post](#3-writing-a-new-post)
4. [Adding a Cover Image to a Post](#4-adding-a-cover-image-to-a-post)
5. [Publishing to GitHub Pages](#5-publishing-to-github-pages)
6. [Day-to-Day Workflow](#6-day-to-day-workflow)
7. [Customising the Site](#7-customising-the-site)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Directory Structure

```
my-blog/
├── hugo.toml                        ← Site config (title, nav, params, social links)
├── content/
│   ├── posts/                       ← All blog posts (Markdown)
│   │   ├── post-template.md         ← Copy this to start a new post
│   │   ├── getting-started.md
│   │   └── hello-world.md
│   └── about.md                     ← About page
├── layouts/
│   ├── _default/
│   │   └── list.html                ← Home grid layout + Posts list (overrides theme)
│   └── partials/
│       └── home_info.html           ← Home hero partial (currently unused)
├── archetypes/
│   └── posts.md                     ← Frontmatter template used by `hugo new content`
├── assets/css/extended/
│   └── custom.css                   ← All CSS customisations (theme, grid, cards)
├── static/
│   └── images/                      ← Put post cover images here
├── themes/PaperMod/                 ← Theme git submodule — do not edit directly
└── .github/workflows/
    └── deploy.yml                   ← Auto-deploys on push to main
```

---

## 2. Local Development

### Prerequisites

Hugo Extended must be installed. After install, open a **new terminal** so it's on your PATH:

```powershell
hugo version
# Expected: hugo v0.14x.x+extended
```

If the command isn't found, use the full path:
```
C:\Users\nagar\AppData\Local\Microsoft\WinGet\Packages\Hugo.Hugo.Extended_Microsoft.Winget.Source_8wekyb3d8bbwe\hugo.exe
```

### Run the dev server

```powershell
cd C:\Users\nagar\my-blog
hugo server -D        # -D includes draft posts
```

Open **http://localhost:1313/blog/** in your browser. The page hot-reloads on every save.

To preview only published (non-draft) posts:
```powershell
hugo server
```

Stop the server: `Ctrl+C`

---

## 3. Writing a New Post

### Option A — Copy the template (recommended)

```powershell
# Copy the template file
copy content\posts\post-template.md content\posts\your-post-slug.md
```

Then open `content/posts/your-post-slug.md` and fill in your content.

### Option B — Use Hugo's generator

```powershell
hugo new content posts/your-post-slug.md
```

This creates a file pre-filled with frontmatter from `archetypes/posts.md`.

### Frontmatter reference

```yaml
---
title: "Your Post Title"
date: 2026-04-03
draft: true                    # Change to false when ready to publish
tags: ["HPC", "AWS"]           # Shown on cards and post pages
categories: ["Infrastructure"] # Broader grouping
description: "One-line summary shown in post list and SEO."
showToc: true                  # Table of contents (false for short posts)
cover:
  image: "images/your-post-cover.png"   # Optional — 1200×630px recommended
  alt: "Description of the image"
  relative: true
---
```

### Summary cutoff

The text before `<!--more-->` is used as the card summary on the home grid.

```markdown
This paragraph appears as the card preview on the home page.

<!--more-->

Everything after here is only visible when the full post is opened.
```

### Publish a post

1. Set `draft: false` in the frontmatter
2. Commit and push:

```powershell
git add content/posts/your-post-slug.md
git commit -m "Publish: Your Post Title"
git push
```

GitHub Actions deploys automatically. Live in ~1 minute.

---

## 4. Adding a Cover Image to a Post

Cover images appear in the home grid cards. Without one, a themed gradient placeholder is shown automatically.

### Steps

1. Place your image in `static/images/` (create the folder if it doesn't exist):
   ```
   static/images/my-post-cover.png
   ```
   Recommended size: **1200 × 630 px** (16:9 aspect ratio works best).

2. Add to the post frontmatter:
   ```yaml
   cover:
     image: "images/my-post-cover.png"
     alt: "Brief description of the image"
     relative: true
   ```

3. The home grid card will show the image instead of the gradient placeholder.

### Free image sources

- [Unsplash](https://unsplash.com) — photography
- [unDraw](https://undraw.co) — tech illustrations (SVG)
- Generate with AI tools for custom visuals

---

## 5. Publishing to GitHub Pages

Already configured. The workflow at `.github/workflows/deploy.yml` builds and deploys automatically on every push to `main`.

### If setting up from scratch

1. Create a GitHub repo (public). Name it `nagarajrs.github.io` for a root URL, or anything else (e.g. `blog`) for `nagarajrs.github.io/blog/`.
2. In repo **Settings → Pages → Source**, select **GitHub Actions**.
3. Update `baseURL` in `hugo.toml` to match your URL.
4. Push to `main` — the workflow handles the rest.

---

## 6. Day-to-Day Workflow

```
1. copy content\posts\post-template.md content\posts\my-new-post.md
2. Write content in the new file
3. hugo server -D          ← preview at localhost:1313/blog/
4. Set draft: false
5. git add content/posts/my-new-post.md
6. git commit -m "Publish: My New Post"
7. git push                ← live in ~1 minute via GitHub Actions
```

---

## 7. Customising the Site

### Site title & social links

Edit `hugo.toml`:

```toml
title = "Nagaraj's Blog"

[params]
  author = "Nagaraj"
  description = "HPC & Cloud Infrastructure | Life Sciences & Pharma | AI/ML at Scale"

[[params.socialIcons]]
  name = "github"
  url = "https://github.com/nagarajrs"

[[params.socialIcons]]
  name = "linkedin"
  url = "https://www.linkedin.com/in/nagaraj-s-54273a1a3/"
```

### Navigation menu

Add or reorder items in `hugo.toml`:

```toml
[[menu.main]]
  identifier = "projects"
  name = "Projects"
  url = "/projects/"
  weight = 4              # Lower = further left
```

Create the page at `content/projects.md` or `content/projects/_index.md`.

### Theme (light / dark / auto)

```toml
[params]
  defaultTheme = "auto"   # auto | light | dark
```

`auto` follows the reader's OS preference and shows the toggle icon in the header.

### Custom CSS

All visual customisation lives in `assets/css/extended/custom.css`. PaperMod merges this automatically. The current theme uses:

| Variable | Value | Purpose |
|---|---|---|
| `--accent` | `#e8961e` | Amber — hover, tags, links |
| `--bg` | `#0a0a0a` | Near-black background |
| `--card-bg` | `#141414` | Post card surface |
| `--text` | `#f0f0f0` | Primary text |
| `--text-muted` | `#888888` | Secondary text |

To change the accent colour, update `--accent` and `--accent-dim` at the top of `custom.css`.

### Post card gradients

When a post has no cover image, the card shows one of 6 gradient placeholders. To customise them, edit the `.grad-0` through `.grad-5` classes in `custom.css`.

### Home grid columns

Controlled by CSS in `custom.css`:
- Mobile (≤680px): 1 column
- Desktop: 2 columns
- Wide (≥1100px): 3 columns

### Code highlight theme

```toml
[markup.highlight]
  style = "dracula"   # monokai | github | nord | solarized-dark | etc.
```

Full list: https://xyproto.github.io/splash/docs/

### Override a theme template

Do **not** edit files inside `themes/PaperMod/`. Instead:

1. Find the file in `themes/PaperMod/layouts/`
2. Copy it to the matching path under the root `layouts/` folder
3. Edit the copy — Hugo prefers root `layouts/` over the theme

---

## 8. Troubleshooting

| Problem | Fix |
|---|---|
| `hugo: command not found` | Open a new terminal — WinGet updates PATH for new sessions only |
| Posts not showing on live site | Check `draft: false` in frontmatter |
| Site broken after push | Check **Actions** tab on GitHub for the error log |
| Links broken on live site | Ensure `baseURL` in `hugo.toml` ends with a trailing slash and matches exactly |
| Theme missing after cloning | Run `git submodule update --init --recursive` |
| Changes not reloading locally | Hard refresh: `Ctrl+Shift+R`. Make sure `hugo server -D` is running |
| Cover image not showing | Ensure `relative: true` is set in frontmatter and file is in `static/images/` |
