---
title: "Hello World 🌍"
date: 2026-04-29T22:00:00+08:00
draft: false
tags: ["Hello", "Blog"]
categories: ["Misc"]
summary: "First post — how and why I set up this blog."
cover:
  image: ""
  alt: ""
  hidden: false
---

## Hello, World! 🎉

Welcome to my blog! This is the very first post, built with [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod).

### Why Hugo?

- ⚡ **Blazing fast** — compiles hundreds of posts in seconds
- 📝 **Markdown-first** — plain text, version-control friendly
- 🎨 **Tons of themes** — active community, great designs
- 🚀 **Easy to deploy** — static files, throw them anywhere

### How I Built It

The whole process was pretty straightforward:

```bash
# 1. Install Hugo
brew install hugo  # macOS
# or grab the deb/rpm

# 2. Create a site
hugo new site my-blog
cd my-blog

# 3. Add a theme
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod

# 4. Write a post
hugo new posts/hello-world.md

# 5. Preview locally
hugo server -D

# 6. Build & deploy
hugo --minify
```

### What's Next

- [ ] Write more tech articles
- [ ] Set up comments
- [ ] Add RSS feed
- [ ] Optimize SEO

Stay tuned! 📡
