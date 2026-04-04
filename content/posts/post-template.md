---
title: "Your Post Title Here"
date: 2026-04-03
draft: true
tags: ["HPC", "AWS", "Kubernetes"]          # 2–4 tags shown on home cards
categories: ["Infrastructure"]              # One broad category
description: "One sentence that summarises this post — shown in SEO and post lists."
showToc: true                               # Set false for short posts
cover:
  image: "images/your-post-cover.png"      # Place file in static/images/
  alt: "Brief description of the image"
  relative: true
  hidden: false                            # true = hide on post page, show only on list
---

<!--
  INSTRUCTIONS
  ─────────────────────────────────────────────────────────────────
  1. Copy this file:  copy post-template.md your-post-slug.md
  2. Fill in the frontmatter above (title, date, tags, description, cover)
  3. Write your post below — the text before <!--more--> is the card summary
  4. Set draft: false when ready to publish
  5. git add, commit, push → live in ~1 minute
  ─────────────────────────────────────────────────────────────────
-->

Write your opening paragraph here. Keep it to 2–3 sentences — this text appears
as the card preview on the home page and in search results.

<!--more-->

Everything after this line is only shown when the full post is opened.

---

## Background

Provide context. Why does this problem exist? Who runs into it?
What environment or scale are we talking about?

---

## The Problem

State the specific issue clearly. What breaks, what's slow, what's missing?
Use concrete numbers where you have them — cluster size, job count, latency, cost.

---

## Approach

Explain your solution approach before diving into details.
A short "here's what we're going to do" paragraph helps readers follow along.

### Step 1 — Set up the environment

Walk through the first step. Include actual commands.

```bash
# Example: start a SLURM cluster
srun --nodes=4 --ntasks-per-node=8 --gres=gpu:2 my_job.sh
```

### Step 2 — Configure X

```yaml
# slurm.conf excerpt
ClusterName=pharma-hpc
SlurmctldHost=head-node
MpiDefault=pmix
ProctrackType=proctrack/cgroup
```

### Step 3 — Run and verify

```python
import subprocess

result = subprocess.run(
    ["squeue", "--format=%i %j %T %R"],
    capture_output=True, text=True
)
print(result.stdout)
```

---

## Results

What did this achieve? Include before/after metrics if you have them.

| Metric | Before | After |
|---|---|---|
| Cluster deploy time | 45 min | 16 min |
| Storage cost / month | $42K | $28K |
| Job queue wait (p95) | 8 min | 2 min |

---

## Key Takeaways

- **Short bold statement.** Explanation of the takeaway in 1–2 sentences.
- **Another takeaway.** What would you do differently, or what generalises beyond this case?
- **Watch out for.** A gotcha or edge case worth noting.

---

## Further Reading

- [Link text](https://example.com) — one-line description of what this is
- [AWS ParallelCluster docs](https://docs.aws.amazon.com/parallelcluster/) — official reference
- Related post: [Title of a future post](#)

---

<!--
  MARKDOWN QUICK REFERENCE
  ─────────────────────────────────────────────────────────────────

  ## Heading 2
  ### Heading 3
  #### Heading 4 (renders as small amber uppercase label in this theme)

  **bold**   *italic*   `inline code`

  > Blockquote text — use for important callouts or quotes

  - Unordered list item
  - Another item
    - Nested item

  1. Numbered list
  2. Second item

  [Link text](https://url.com)

  ![Image alt](images/filename.png)

  ```python
  # Fenced code block with language for syntax highlighting
  # Supported: bash, python, go, yaml, toml, json, hcl, c, cpp, sql, etc.
  ```

  | Column 1 | Column 2 | Column 3 |
  |---|---|---|
  | Cell | Cell | Cell |

  ---   (horizontal rule / section divider)

  ─────────────────────────────────────────────────────────────────
-->
