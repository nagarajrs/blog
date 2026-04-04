# Nagaraj's Blog — Setup & Publishing Guide

Everything you need to go from zero to a live blog post on GitHub Pages.

---

## Table of Contents

1. [One-Time Setup](#1-one-time-setup)
2. [Preview Locally](#2-preview-locally)
3. [Publish to GitHub Pages](#3-publish-to-github-pages)
4. [Writing a New Post](#4-writing-a-new-post)
5. [Editing Existing Content](#5-editing-existing-content)
6. [Day-to-Day Workflow](#6-day-to-day-workflow)
7. [Customising the Site](#7-customising-the-site)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. One-Time Setup

### A. Open a new terminal

Hugo was just installed. Open a **new** PowerShell or Command Prompt window so that `hugo` is on your PATH. Verify:

```powershell
hugo version
# Expected: hugo v0.159.x-... extended windows/amd64
```

If you still get "command not found", use the full path for now:

```
C:\Users\nagar\AppData\Local\Microsoft\WinGet\Packages\Hugo.Hugo.Extended_Microsoft.Winget.Source_8wekyb3d8bbwe\hugo.exe
```

### B. Navigate to the blog folder

```powershell
cd C:\Users\nagar\my-blog
```

### C. Update your personal details in `hugo.toml`

Open `hugo.toml` and change these two lines to match your actual GitHub username:

```toml
baseURL = "https://YOUR-GITHUB-USERNAME.github.io/"

[[params.socialIcons]]
  name = "github"
  url = "https://github.com/YOUR-GITHUB-USERNAME"
```

Also update your name and description if needed:

```toml
[params]
  author = "Your Name"
  description = "Your blog description."

[params.homeInfoParams]
  Title = "Hi, I'm Your Name"
  Content = "Your welcome message."
```

---

## 2. Preview Locally

Run the development server from inside `C:\Users\nagar\my-blog`:

```powershell
hugo server -D
```

- `-D` includes draft posts (so you can preview before publishing)
- Open your browser at **http://localhost:1313**
- The page live-reloads whenever you save a file — no need to restart

To stop the server: press `Ctrl+C`.

To preview only published posts (no drafts):

```powershell
hugo server
```

---

## 3. Publish to GitHub Pages

Do this once to connect the blog to GitHub. After that, publishing is just a `git push`.

### Step 1 — Create a GitHub repository

1. Go to [github.com/new](https://github.com/new)
2. Name it **`YOUR-GITHUB-USERNAME.github.io`**
   - Example: if your username is `nagaraj7`, name it `nagaraj7.github.io`
   - This gives you the URL `https://nagaraj7.github.io/` for free
3. Set it to **Public**
4. Do **not** add a README or .gitignore (the repo should be empty)
5. Click **Create repository**

> **Alternative**: You can name the repo anything (e.g., `blog`). Your URL will then be `https://USERNAME.github.io/blog/`. If you do this, update `baseURL` in `hugo.toml` to include the `/blog/` suffix.

### Step 2 — Enable GitHub Pages via Actions

1. In your new repo, go to **Settings → Pages**
2. Under **Source**, select **GitHub Actions**
3. Click **Save**

### Step 3 — Push the blog to GitHub

In `C:\Users\nagar\my-blog`, run:

```powershell
git remote add origin https://github.com/YOUR-GITHUB-USERNAME/YOUR-REPO-NAME.git
git add .
git commit -m "Initial blog setup"
git push -u origin main
```

### Step 4 — Watch it deploy

1. Go to your repo on GitHub
2. Click the **Actions** tab
3. You'll see a workflow called **"Deploy to GitHub Pages"** running
4. Wait ~1 minute for it to complete (green checkmark)
5. Visit `https://YOUR-GITHUB-USERNAME.github.io/` — your blog is live

> The deployment workflow runs automatically on every future `git push` to `main`.

---

## 4. Writing a New Post

### Step 1 — Create the file

From `C:\Users\nagar\my-blog`, run:

```powershell
hugo new content posts/my-post-title.md
```

Replace `my-post-title` with a URL-friendly name using hyphens (e.g., `hugo new content posts/how-i-use-go-channels.md`).

This creates the file at `content/posts/my-post-title.md` pre-filled with frontmatter from the archetype.

### Step 2 — Edit the frontmatter

Open the file. The top section between `---` lines controls metadata:

```yaml
---
title: "My Post Title"          # Displayed title (can have spaces and capitals)
date: 2026-04-03                # Publication date
draft: true                     # CHANGE TO false when ready to publish
tags: ["go", "systems"]         # Optional: for filtering/discovery
categories: ["Engineering"]     # Optional: broader grouping
description: "A one-line summary shown in the post list."
showToc: true                   # Show table of contents (set false for short posts)
---
```

**Important**: while `draft: true`, the post is invisible on the live site. Change it to `draft: false` only when you're ready to publish.

### Step 3 — Write your post

Below the closing `---`, write in plain Markdown:

```markdown
Opening paragraph that appears in the post list preview.

<!--more-->

Everything after this line is hidden from the list — only visible when the full post is opened.

## Section Heading

Regular paragraph text.

### Subsection

- Bullet point
- Another point

**Bold text**, *italic text*, `inline code`.

[Link text](https://example.com)
```

### Step 4 — Add code blocks

Wrap code with triple backticks and a language name for syntax highlighting:

````markdown
```go
func main() {
    fmt.Println("Hello, World!")
}
```

```python
def greet(name):
    return f"Hello, {name}"
```

```bash
git push origin main
```
````

Supported language names: `go`, `python`, `javascript`, `typescript`, `bash`, `sql`, `yaml`, `toml`, `json`, `rust`, `c`, `cpp`, `java`, `html`, `css`, and many more.

### Step 5 — Preview it

```powershell
hugo server -D
```

Visit http://localhost:1313 and check your post looks right.

### Step 6 — Publish

1. Set `draft: false` in the frontmatter
2. Push to GitHub:

```powershell
git add content/posts/my-post-title.md
git commit -m "Publish: My Post Title"
git push
```

GitHub Actions deploys automatically. Live in ~1 minute.

---

## 5. Editing Existing Content

### Edit a post

Open the file directly (e.g., `content/posts/getting-started.md`) in any text editor, make your changes, save, then push:

```powershell
git add content/posts/getting-started.md
git commit -m "Update getting-started post"
git push
```

### Edit the About page

Open `content/about.md`, edit the Markdown content, then push.

### Change site title, description, or social links

Edit `hugo.toml`. The most common fields to update:

| Field | What it changes |
|---|---|
| `title` | Browser tab and site header |
| `[params] description` | Meta description (SEO) |
| `[params.homeInfoParams] Title` | Hero heading on homepage |
| `[params.homeInfoParams] Content` | Hero subtext on homepage |
| `[params.socialIcons] url` | GitHub profile link |
| `baseURL` | Your live site URL |

---

## 6. Day-to-Day Workflow

Once the site is live, writing a new post is:

```
1. hugo new content posts/post-name.md
2. Edit the file, write your content
3. hugo server -D  (preview locally)
4. Set draft: false
5. git add . && git commit -m "Publish: Post Name" && git push
```

That's it. GitHub Actions handles the build and deploy.

---

## 7. Customising the Site

### Change the colour theme

In `hugo.toml`:

```toml
[params]
  defaultTheme = "dark"   # Options: "auto" | "light" | "dark"
```

`auto` follows the reader's OS setting.

### Change the code highlight colour scheme

In `hugo.toml`:

```toml
[markup.highlight]
  style = "dracula"   # Change to: monokai, github, nord, solarized-dark, etc.
```

Full list of styles: https://xyproto.github.io/splash/docs/

### Custom CSS

Edit `assets/css/extended/custom.css` to override any styles. PaperMod merges this file automatically. Example:

```css
/* Increase font size */
body {
  font-size: 18px;
}
```

### Add a new menu item

In `hugo.toml`, add another `[[menu.main]]` block:

```toml
[[menu.main]]
  identifier = "projects"
  name = "Projects"
  url = "/projects/"
  weight = 4
```

Then create the page at `content/projects.md` or `content/projects/_index.md`.

### Override a theme template

Do **not** edit files inside `themes/PaperMod/`. Instead:

1. Find the template in `themes/PaperMod/layouts/`
2. Copy it to the same relative path under the root `layouts/` folder
3. Edit the copy — Hugo will use it instead of the theme version

---

## 8. Troubleshooting

**`hugo: command not found` after install**
Open a new terminal window. WinGet adds Hugo to your PATH, but the current session doesn't see it yet.

**Posts not showing on the live site**
Check that `draft: false` is set in the post's frontmatter. Drafts never appear on the live site.

**Site looks broken after pushing**
Check the **Actions** tab on GitHub. If the workflow failed, click it to see the error log.

**`baseURL` is wrong and links are broken**
Update `baseURL` in `hugo.toml` to exactly match your GitHub Pages URL (including the trailing slash), then push again.

**Theme missing after cloning on another machine**
The theme is a git submodule. After cloning, run:
```powershell
git submodule update --init --recursive
```

**Changes not appearing locally**
Make sure `hugo server -D` is running. If it is, try a hard refresh in the browser (`Ctrl+Shift+R`).

---

## Directory Reference

```
my-blog/
├── hugo.toml                  ← Site config (title, theme, menus, params)
├── content/
│   ├── posts/                 ← Blog posts go here
│   │   ├── getting-started.md
│   │   └── hello-world.md
│   └── about.md               ← About page
├── archetypes/
│   └── posts.md               ← Template used by `hugo new content`
├── assets/css/extended/
│   └── custom.css             ← Your CSS overrides
├── themes/PaperMod/           ← Theme (git submodule, do not edit)
├── static/                    ← Images and other static files
└── .github/workflows/
    └── deploy.yml             ← Auto-deploy to GitHub Pages on push
```
