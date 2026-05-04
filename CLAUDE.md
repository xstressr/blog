# JOJO's Blog

Hugo blog using PaperMod theme, deployed to Cloudflare Pages.

## Key Info

- **Framework**: Hugo v0.147.1 (extended)
- **Theme**: PaperMod (git submodule at `themes/PaperMod`)
- **Domain**: `blog.crazyai.uk`
- **Deployment**: Cloudflare Pages — push to `master` auto-deploys
- **Language**: English only — all posts, titles, descriptions must be in English

## Build & Dev

```bash
# Init submodules (required for fresh clone or CI)
git submodule update --init --recursive

# Local dev server
hugo server -D          # http://localhost:1313, includes drafts

# Production build
hugo --minify           # output → public/
```

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
---
```

### Rules

1. `draft: false` must be set — Hugo skips drafts by default
2. Do NOT use future dates — Hugo hides future posts. Use today's date or past
3. Timezone: `+08:00` (Asia/Shanghai)
4. All content in English

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

| File | Title | Date |
|------|-------|------|
| `hello-world.md` | Hello World | 2026-04-29 |
| `picgen-devops-day.md` | A Day of DevOps | 2026-05-02 |
| `picgen-studio-chat-style-image-creation.md` | Building a Chat-Style Image Creation Studio | 2026-05-02 |

## Pitfalls

- **Git submodule on CF Pages**: Build command MUST include `git submodule update --init --recursive`
- **Future posts**: Hugo skips them. Use current or past dates
- **Bot Fight Mode**: Cloudflare may block `curl`/bot requests with 403. Site works fine in browsers
- **`public/` is gitignored**: CF Pages builds from source, don't commit build output
