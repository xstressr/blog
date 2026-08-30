# Zero to One AI

Hugo blog using PaperMod theme, deployed to Cloudflare Pages.

## Key Info

- **Framework**: Hugo v0.147.1 (extended)
- **Theme**: PaperMod (git submodule at `themes/PaperMod`)
- **Domain**: `blog.crazyai.uk`
- **Deployment**: Cloudflare Pages — push to `master` auto-deploys
- **Editorial direction**: a public zero-to-one AI learning log built around experiments, mistakes, and clear explanations
- **Language**: English by default — keep navigation and core site copy in English

## Build & Dev

Hugo is NOT installed locally — do not attempt `hugo server` or `hugo --minify`. Cloudflare Pages handles all builds. Just write content and push.

## Writing Posts

- **Location**: `content/posts/<post-name>.md`
- **Naming**: lowercase, hyphens, no spaces

### Frontmatter

```markdown
---
title: "Post Title"
date: 2026-05-04T20:00:00+08:00
summary: "Short summary"
categories: ["Tech"]
tags: ["tag1", "tag2"]
draft: false
# math: true  # optional; loads KaTeX for this post
---
```

### Rules

1. `draft: false` must be set — Hugo skips drafts by default
2. Do NOT use future dates — Hugo hides future posts. Use today's date or past
3. Timezone: `+08:00` (Asia/Shanghai)
4. Prefer English for public posts unless the author explicitly chooses a Chinese learning note
5. Do not frame the site around the author's previous industry or job title; the subject is the AI learning journey itself
6. Add `math: true` when a post contains LaTeX. Use `\\(inline\\)` and `$$display$$` delimiters.
7. Folio series posts set `note`, `series`, and `plate` (for example `"I / IV"`). Pair `folio-card` shortcodes. State evidence boundaries: what ran, what was only read.

## Deploy

```bash
git add content/posts/new-post.md
git commit -m "new post: post title"
git push    # Cloudflare Pages auto-deploys
```

## Git Identity

- Name: `jojo bot`
- Email: `xneroial@gmail.com`

## Existing Posts

Keep this table in date order when adding a post. `note` numbers are explicit frontmatter, not automatic.

| File | Title | Date | note |
|------|-------|------|------|
| `hello-world.md` | Hello World | 2026-04-29 | 01 |
| `picgen-devops-day.md` | A Day of DevOps | 2026-05-02 | 02 |
| `picgen-studio-chat-style-image-creation.md` | Building a Chat-Style Image Creation Studio | 2026-05-02 | 03 |
| `picgen-prompt-examples-and-reference-image-entry.md` | From Blank Canvas to Guided Creation | 2026-05-04 | 04 |
| `transformer-field-notes-01-batch-factory-of-futures.md` | The Batch Is a Factory of Futures | 2026-08-20 | 05 |
| `transformer-field-notes-02-four-ways-to-average-the-past.md` | Four Ways to Average the Past | 2026-08-20 | 06 |
| `transformer-field-notes-03-residual-stream-is-the-model.md` | The Residual Stream Is the Model | 2026-08-20 | 07 |
| `transformer-field-notes-04-live-wire-is-not-a-language.md` | A Live Wire Is Not a Language | 2026-08-20 | 08 |
| `transformer-field-notes-05-width-paid-context-lied.md` | Width Paid, Context Lied, Last Step Lost | 2026-08-20 | 09 |
| `transformer-field-notes-06-extra-code-is-not-another-brain.md` | The Extra Code Is Not Another Brain | 2026-08-20 | 10 |
| `inference-field-notes-01-the-model-file-has-no-names.md` | The Model File Has No Names | 2026-08-22 | 11 |
| `c-field-notes-01-python-writes-the-answer-sheet.md` | Python Writes the Answer Sheet | 2026-08-27 | 12 |
| `c-field-notes-02-there-is-no-autograd-here.md` | There Is No Autograd Here | 2026-08-27 | 13 |
| `adapter-field-notes-01-the-loss-is-not-the-task.md` | The Loss Is Not the Task | 2026-08-30 | 14 |
| `adapter-field-notes-02-the-mask-is-the-objective.md` | The Mask Is the Objective | 2026-08-30 | 15 |
| `adapter-field-notes-03-scale-bought-the-base.md` | Scale Bought the Base | 2026-08-30 | 16 |

## Pitfalls

- **Git submodule on CF Pages**: Build command MUST include `git submodule update --init --recursive`
- **Future posts**: Hugo skips them. Use current or past dates
- **Bot Fight Mode**: Cloudflare may block `curl`/bot requests with 403. Site works fine in browsers
- **`public/` is gitignored**: CF Pages builds from source, don't commit build output
