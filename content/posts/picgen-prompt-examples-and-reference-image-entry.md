---
title: "From Blank Canvas to Guided Creation — Prompt Examples and Reference Image Entry in PicGen Studio"
date: 2026-05-04T22:00:00+08:00
draft: false
note: "04"
tags: ["picgen", "ai", "product-design", "nextjs", "fastapi", "ux"]
categories: ["tech"]
summary: "PicGen Studio now has Prompt Examples that fill prompts with one click, and a proper 'Add Reference' button that lets you upload local images or pick from history. No more staring at a blank input box."
---

## The Blank Canvas Problem

When I built the Studio last week, the conversational interface worked well for iterative creation. But there was still a friction point: every conversation started with a blank prompt box and zero inspiration.

The admin panel already had a Prompt Example system — curated prompts with showcase images, categories, and tags. But it was completely invisible to users on the generation pages. Text to Image had no examples at all. Image to Image showed examples at the bottom of the page, below the form. Studio had nothing.

The result? Users had to write every prompt from scratch. Meanwhile, the admin team was maintaining a growing collection of reusable prompts that nobody was using.

## The Fix: Three Changes in One Day

### 1. Prompt Examples Everywhere

The biggest change was wiring Prompt Examples into all three generation surfaces.

**Text to Image** now loads `text2img` examples at the top of the page. Each card shows the showcase image, title, category, and tags. Clicking "Use Prompt" fills the prompt and size fields immediately. Since text-to-image doesn't need a reference image, clicking an example and hitting generate just works.

**Image to Image** already had examples, but they were buried at the bottom. The bigger semantic fix: Prompt Examples are now clearly labeled as prompt recipes, not image references. The example card fills the prompt, but you still need to upload your own image. No more confusion about whether clicking an example uses the official showcase image as input.

**Studio** got a new Prompt Example picker integrated into the composer area. A small icon lets you browse and select examples. Text2img examples fill the prompt directly. For img2img examples, the system checks whether you have an active reference image — if not, it prompts you to add one first.

The key design rule that runs through everything:

```
Prompt Example = reusable prompt recipe + showcase image
Reference image = user upload, history image, or active Studio image
```

These two concepts were previously conflated. Separating them made the product logic much cleaner.

### 2. Studio Reference Image Entry

The Studio's `+` button used to be confusing. It looked like "add image" but actually meant "clear reference and go back to text-to-image mode." Clicking it on an empty conversation did nothing — which felt broken.

The redesign turns `+` into a proper "Add Reference Image" entry point. Clicking it opens a bottom sheet with two tabs:

**Upload tab**: Select a local image (PNG/JPG/WEBP), see an instant preview. The image stays as a pending reference on the frontend — no upload to the server yet, no credits charged, no history record created. When you send your next prompt, the image goes as a multipart form upload alongside the text. The backend processes it as img2img, and the result becomes the new active image for subsequent iterations.

**History tab**: Browse your completed generations, paginated. Selecting a history image sets it as the active image via the existing API. If you don't have a conversation yet, one is created automatically.

This distinction matters: uploading a reference image is an input action, not a generation action. It shouldn't create history records or charge credits until you actually generate something.

### 3. Admin Panel Polish

The admin side got validation improvements. Creating a Prompt Example now enforces: title required, category required, prompt required, type must be `text2img` or `img2img`, size must be in the allowed list. Publishing requires a showcase image — you can save a draft without one, but can't go live.

The edit dialog was also reworked: viewport-aligned with internal scrolling, single-column layout on narrow screens, horizontal scroll on data tables to prevent layout breakage.

## The UX Review Document

Before writing code, I spent time on a thorough UX review document (`docs/prompt-example-user-experience-review.md`). It analyzed the current state across all three generation pages, referenced an external gallery site for inspiration, and laid out a phased rollout plan.

The external reference (a GPT-Image-2 gallery) taught an important workflow lesson:

```
Gallery-first: Browse examples → choose a close case → adapt prompt → generate
vs.
Form-first: Choose mode → face blank form → write prompt from scratch
```

PicGen was firmly in the "form-first" camp. The Prompt Example integration shifts it toward a gallery-first experience where possible, while keeping the form available for users who know exactly what they want.

## The Reference Image Design Document

The Studio reference image entry also got its own design doc (`docs/studio-reference-image-entry-design.md`). It covers the full flow: why the `+` button was broken, what "add reference" should mean, how pending uploads work without creating history records, the new multipart API endpoint, and a four-phase rollout plan.

The design explicitly avoids several tempting but problematic approaches:

- **Don't create history records for uploads** — an upload is input material, not a generation result. Mixing them pollutes the history page.
- **Don't use Prompt Example showcase images as hidden references** — this was the old behavior that created semantic confusion.
- **Don't persist uploaded references across refreshes in V1** — keep the model simple. Upload, generate, done.

## Technical Bits

A few smaller fixes landed alongside the features:

- **Font consistency**: Updated the `font-sans` CSS variable to use `geist-sans` across the app.
- **Frontend compatibility guidelines**: Added responsive UI guidelines to `AGENTS.md` so that agent-assisted development respects mobile layout constraints.
- **i18n**: All new UI strings have both English and Chinese translations.

The new `reference-picker.tsx` component (308 lines) handles the upload/history tabs. The `studio-example-picker.tsx` (213 lines) handles Prompt Example browsing in Studio. The admin page refactor touched 441 lines of changes.

## What's Next

The Prompt Example system has a clear roadmap:

- **Phase 1** (done): Correct the semantics — examples are prompts, not hidden image inputs.
- **Phase 2** (done today): Admin panel productization + examples on all generation pages.
- **Phase 3** (upcoming): Homepage gallery, deep linking with `?example={id}`, shareable prompt URLs.
- **Phase 4** (future): Usage metrics — copy count, use count, generation count per example.

For Studio reference images, the next phase is Prompt Example + reference image interaction: when you select an img2img example without having a reference image, the system should auto-open the "Add Reference" sheet instead of showing an error.

---

*PicGen is at `pic.crazyai.uk`. Stack: FastAPI + Next.js 16 + PostgreSQL + Cloudflare R2, deployed via Docker Compose. Today's changes spanned ~2,500 lines across 21 files — most of it was frontend work. The design documents took almost as long as the code.*
