---
title: "Building a Chat-Style Image Creation Studio — From Form-Based Generation to Conversational AI Art"
date: 2026-05-02T23:00:00+08:00
draft: false
note: "03"
tags: ["picgen", "ai", "product-design", "nextjs", "fastapi"]
categories: ["tech"]
summary: "PicGen's image generation used to be a simple form: type a prompt, get an image. I just shipped a Studio that turns the whole experience into a ChatGPT-like conversation — where you talk to an AI artist, iterate on images, and build a creative thread."
---

## The Problem with Forms

PicGen has had text-to-image and image-to-image generation for a while. The workflow was straightforward:

1. Pick a mode (text2img or img2img)
2. Type a prompt
3. Optionally select a reference image
4. Click "Generate"
5. See the result

It worked. But it felt... transactional. Like filling out a tax form that happens to produce pictures.

Every generation was a one-shot interaction. Want to iterate on a result? Go to history, find the image, click "continue creation," get redirected to the image-to-image page, re-read your old prompt, tweak it, generate again. Each step was a context switch. The creative flow kept getting interrupted.

What I actually wanted was something like this:

```
Me: Draw a cute cartoon cat in a yellow raincoat
AI: [generates image] 
Me: Nice! Now make it raining, and give the cat a cup of hot cocoa
AI: [generates new image based on the previous one]
Me: The raincoat should be more transparent, you can see the pattern underneath
AI: [generates again]
```

A conversation. Not a form.

## Designing the Studio

The core idea was simple: wrap the existing image generation pipeline in a ChatGPT-like interface. The backend already supports everything needed — text-to-image, image-to-image with a source image, and history-based continuation. What was missing was the conversational layer on top.

### The Key Insight: active_image_id

The whole design hinges on one concept: **the active image**.

In a conversation, when you send a new message, the system needs to know: should this be a fresh generation (text-to-image), or should it build on an existing image (image-to-image)?

The rule is dead simple:

```
No active image → text-to-image
Has active image → image-to-image, using that image as reference
```

After any image is generated, it automatically becomes the active image. You can also manually pick any completed image in the conversation as the active one ("continue with this"), or clear it entirely to start fresh ("new image").

This state lives in the backend (`image_conversations.active_image_id`), so it survives page refreshes and device switches.

### What We're NOT Building (Yet)

V1 is deliberately minimal. We explicitly scoped out:

- **No automatic intent detection** — the system doesn't try to guess if you want text2img or img2img. The active image state handles this implicitly.
- **No version tree visualization** — no branching, no DAG view. It's a linear timeline.
- **No local inpainting** — no masks, no brush tools.
- **No real-time streaming** — images still generate asynchronously with polling.
- **No prompt summarization** — each message is a standalone prompt.

The goal was to ship a working conversational experience, not to boil the ocean.

## The Data Model

Three new pieces in the database:

### image_conversations

```
id, user_id, title, active_image_id, default_model, default_size, created_at, updated_at
```

A conversation groups everything together. Title auto-generates from the first prompt (first 30 characters).

### image_messages

```
id, conversation_id, user_id, role, content, image_id, status, error_message, created_at
```

Messages in the timeline. Two roles: `user` (text prompts) and `assistant` (generated images). Each assistant message links to an `image_id`.

### images table extension

Added `conversation_id` and `source_image_id` columns. `conversation_id` links images to their conversation. `source_image_id` records the parent image in an image-to-image chain — this was already working for history-based continuation, now it's formally part of the schema.

The relationship looks like:

```
ImageConversation
  └── Messages (user text + assistant image references)
        └── Images (with source_image_id chains)
```

## The API

Five new endpoints, keeping it clean:

| Endpoint | Purpose |
|----------|---------|
| `POST /api/image-conversations` | Create a new conversation |
| `GET /api/image-conversations/{id}` | Get conversation with all messages |
| `POST /api/image-conversations/{id}/messages` | Send a message and trigger generation |
| `PATCH /api/image-conversations/{id}/active-image` | Set or clear the active image |
| `POST /api/image-conversations/from-image/{image_id}` | Bootstrap a conversation from a history image |

The message endpoint is the heart of it. When you send a message:

1. Create a user message record
2. Determine generation mode (auto/text2img/img2img)
3. Queue the image generation (reusing existing pipeline)
4. Create an assistant message linked to the new image
5. Return immediately — frontend polls for completion

The backend reuses 100% of the existing generation pipeline. No new provider calls, no new image processing logic. `queue_text2img()` and `queue_img2img()` got a new `conversation_id` parameter, that's it.

## The Frontend

### Studio Page (`/studio`)

The main page is clean — a conversation timeline, an active image panel, and a composer at the bottom.

**ConversationTimeline** renders messages as bubbles. User messages show text. Assistant messages show the generated image (or a loading spinner while generating). Each completed image has a "Continue with this" button.

**ActiveImagePanel** shows the current reference image at the top. It's a constant reminder of "what you're iterating on." You can clear it to switch back to text-to-image mode.

**StudioComposer** is the input area. Type your prompt, hit send. Simple as that.

### The Polling Flow

Still using the same polling strategy as before — no WebSockets in V1:

```
Send message → POST /messages → get image_id
  → Poll GET /images/{id} every 2 seconds
  → On completion: refresh GET /conversations/{id}
  → Timeline updates with final image
```

The 3-minute timeout handles cases where generation takes unusually long.

### History Integration

The history page got a small but important update. The "Continue creation" button now creates a conversation from that image and redirects to Studio, instead of jumping to the old image-to-image form page.

```
Old: History → /generate/image?source={id} → form
New: History → POST /from-image/{id} → /studio?conversation={id}
```

The old form page still works for power users who want direct control. Studio is the new default path.

## Testing

14 backend test cases covering:

- Conversation CRUD
- Auto mode routing (text2img when no active image, img2img when there is one)
- Permission checks (can't use someone else's image as active)
- Status validation (can't use pending/failed images)
- Credit rollback on insufficient balance
- History-to-conversation bootstrapping

All passing. Frontend typechecks and lints clean.

## What's Next

V1 is the foundation. The conversational container is solid, and the image generation pipeline is unchanged. V2 territory includes:

- **Prompt summarization** — auto-generating better prompts from conversation context
- **Branching** — fork from any point in the conversation to explore different directions
- **Multi-candidate generation** — generate 2-3 variants and pick the best one
- **Natural language intent** — "make it more dramatic" without explicit prompt engineering

But those are all additive. The hard part — building the conversation infrastructure, threading images through messages, keeping state consistent across refreshes — that's done.

## Try It

If you have access to PicGen at `pic.crazyai.uk`, look for **Studio** in the sidebar. Start typing. The first message creates a conversation automatically. From there, just keep talking to the AI artist.

It feels surprisingly natural. Much better than filling out forms.

---

*PicGen is self-hosted on a VPS behind Cloudflare. Stack: FastAPI + Next.js 16 + PostgreSQL, all in Docker Compose. The Studio feature took about a day of focused work — most of the time went into designing the data model and conversation flow, not the actual coding.*
