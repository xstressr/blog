# Blog Knowledge Base — JOJO's Blog

## Overview

- **Framework**: Hugo v0.147.1 (extended)
- **Theme**: PaperMod (git submodule)
- **GitHub**: `github.com/xstressr/blog` (branch: `master`)
- **Domain**: `blog.crazyai.uk`
- **Deployment**: Cloudflare Pages (project name: `blog`, default subdomain: `blog-e3c.pages.dev`)
- **Local directory**: `/root/blog/`
- **Content language**: English only — all posts, titles, descriptions must be in English

## Architecture

```
/root/blog/
├── hugo.toml                  # Main config
├── content/
│   ├── posts/                 # Blog posts (Markdown)
│   │   ├── hello-world.md
│   │   ├── picgen-devops-day.md
│   │   └── picgen-studio-chat-style-image-creation.md
│   ├── archives.md            # Archives page
│   └── search.md              # Search page
├── themes/
│   └── PaperMod/              # Theme (git submodule)
├── public/                    # Build output (gitignored)
└── .gitignore
```

## Build & Deploy

### Build command
```bash
cd /root/blog
git submodule update --init --recursive && hugo --minify
```

### Cloudflare Pages build settings
- **Build command**: `git submodule update --init --recursive && hugo --minify`
- **Output directory**: `public`
- **Environment variable**: `HUGO_VERSION = 0.147.1`

### Deploy workflow
Push to `master` on GitHub → Cloudflare Pages auto-deploys. No manual step needed.

```bash
git add .
git commit -m "new post: title here"
git push
```

### Git identity
- Name: `jojo bot`
- Email: `xneroial@gmail.com`

## Hugo Configuration (`hugo.toml`)

Key settings:

| Setting | Value |
|---------|-------|
| `baseURL` | `https://blog.crazyai.uk/` |
| `languageCode` | `en` |
| `defaultContentLanguage` | `en` |
| `title` | `JOJO's Blog` |
| `theme` | `PaperMod` |
| `defaultTheme` | `auto` (follows system) |
| `enableEmoji` | `true` |
| `enableRobotsTXT` | `true` |
| `enableGitInfo` | `true` |
| `ShowShareButtons` | `true` |
| `ShowReadingTime` | `true` |
| `ShowWordCount` | `true` |
| `ShowCodeCopyButtons` | `true` |
| `UseHugoToc` | `true` |
| Code highlight style | `monokai` |
| Outputs (home) | `HTML`, `RSS`, `JSON` |

Social icons: GitHub → `https://github.com/xstressr`

Menu: Posts, Tags, Archives, Search, About

## Writing a New Post

### Frontmatter template

```markdown
---
title: "Your Post Title"
date: 2026-05-04T20:00:00+08:00
summary: "Short summary for the post list"
categories: ["Tech"]
tags: ["tag1", "tag2"]
draft: false
---

## Introduction

Content here...
```

### Important rules

1. **Language**: All content must be in English
2. **Date**: Do NOT use future dates — Hugo skips future posts by default. Use the current date or past dates.
3. **Timezone**: Use `+08:00` (Asia/Shanghai)
4. **`draft: false`**: Must be explicitly set, otherwise Hugo won't publish it
5. **File location**: `/root/blog/content/posts/your-post-name.md`
6. **File naming**: lowercase, hyphens, no spaces (e.g., `my-cool-post.md`)

### Local preview

```bash
cd /root/blog
hugo server -D    # -D includes drafts
# Visit http://localhost:1313
```

### Publish

```bash
cd /root/blog
hugo --minify                           # Verify build succeeds
git add content/posts/new-post.md
git commit -m "new post: post title"
git push                                # CF Pages auto-deploys
```

## Existing Posts

| File | Title | Date | Tags |
|------|-------|------|------|
| `hello-world.md` | Hello World | 2026-04-29 | blog, hello |
| `picgen-devops-day.md` | A Day of DevOps | 2026-05-02 | docker, devops, troubleshooting |
| `picgen-studio-chat-style-image-creation.md` | Building a Chat-Style Image Creation Studio | 2026-05-02 | ai, picgen, fastapi, nextjs, product-design |

## Cloudflare Configuration

| Item | Value |
|------|-------|
| Zone | `crazyai.uk` |
| Zone ID | `92825ba7941c88c2aa6be2e388aab0c2` |
| Account ID | `1cde6a4c9c376eab47d106fdb2151202` |
| CF Pages project | `blog` |
| Pages default URL | `blog-e3c.pages.dev` |
| Custom domain | `blog.crazyai.uk` |
| Plan | Free |

## Known Issues & Pitfalls

1. **Git submodule on CF Pages**: Cloudflare Pages does NOT clone submodules by default. Build command MUST include `git submodule update --init --recursive && hugo --minify`.

2. **Future posts hidden**: Hugo skips posts with future dates. Either use today's date or pass `--buildFuture` flag.

3. **Bot Fight Mode 403**: Cloudflare's Bot Fight Mode can block `curl`/bot requests, returning 403 on `blog.crazyai.uk`. The site works fine in browsers. This is a WAF-level setting, not a blog issue.

4. **`public/` is gitignored**: The build output directory is not committed to git. CF Pages builds it from source.

5. **PaperMod as submodule**: The theme lives at `themes/PaperMod` as a git submodule. If cloning the repo fresh, always run `git submodule update --init --recursive`.

## Fresh Clone & Setup

```bash
git clone https://github.com/xstressr/blog.git
cd blog
git submodule update --init --recursive

# Install Hugo if needed
# Hugo extended version required for PaperMod
hugo version    # should be 0.147.1+

# Local dev
hugo server -D

# Build
hugo --minify
```

## Related Projects

- **PicGen** (`/root/picgen/`): AI image generation platform at `pic.crazyai.uk`. Blog posts reference this project.
- **Hermes Agent**: AI agent that manages the blog from Telegram/Discord.
