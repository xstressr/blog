---
title: "A Day of DevOps: Docker Permission Hell, Disk Cleanup, and the Turnstile Mystery"
date: 2026-05-02T20:00:00+08:00
draft: false
tags: ["devops", "docker", "picgen", "troubleshooting"]
categories: ["tech"]
summary: "What started as a simple Docker rebuild turned into a full-day adventure — fighting uv venv permissions, freeing 13GB of disk, and debugging a Cloudflare Turnstile secret that refused to work."
---

## The Morning Situation

It was supposed to be a quick task: rebuild the PicGen Docker containers with the latest environment variables. PicGen is my self-hosted AI image generation platform (think DALL-E but self-managed), running on a small VPS behind Cloudflare at `pic.crazyai.uk`.

Simple, right? Just `docker compose down && docker compose up -d`. In and out, 5 minutes.

Famous last words.

## Act 1: The Disk Space Crisis

Before I could even start the build, I hit the first wall:

```
ERROR: failed to solve: write /app/.venv/lib/python3.11/site-packages/...: no space left on device
```

Disk usage: **34G / 40G (91%)**. On a 40GB VPS, that's basically full.

A quick `du -sh` investigation revealed the culprit: **OpenClaw**, a previous AI agent framework I'd been experimenting with, was sitting at **9.1GB** in `~/.openclaw/`. I'd already migrated to Hermes Agent weeks ago, but never got around to cleaning up the old installation.

### The Cleanup

First, safety — I had an Obsidian vault in there with notes. Quick git commit and push:

```bash
cd ~/.openclaw/workspace/notes/obsidian-vault
git add -A && git commit -m "backup before openclaw cleanup" && git push
```

Then the teardown:

```bash
# Stop the systemd service
systemctl --user disable --now openclaw-gateway.service

# Remove the service file
rm ~/.config/systemd/user/openclaw-gateway.service

# Delete the 9.1GB directory
rm -rf ~/.openclaw

# Uninstall the CLI (442 packages!)
npm rm -g openclaw
```

But I wasn't done. A broader sweep found more junk:

- **npm cache**: 2.5GB
- **`.cache/` directory**: 4.1GB (camoufox, uv, pnpm, pip caches)
- **`/tmp` artifacts**: ~280MB of old Playwright downloads

Total freed: **~13GB**. Disk went from 91% to 54%. Breathing room restored.

## Act 2: Docker Permission Hell

With disk space available, I ran the build. The image built fine. The container started. And then:

```
PermissionError: [Errno 13] Permission denied: '/app/.venv/bin/python3'
```

This one took a while to figure out. Here's what happened:

### The Root Cause

The PicGen API Dockerfile had been updated to use `uv` (the fast Python package manager from Astral) instead of `pip`. The `uv sync` command creates a virtual environment, and by default it symlinks the Python binary to `~/.local/share/uv/python/`.

The problem? In the Docker container, `~` resolves to `/root/` during build (running as root), but the app runs as `appuser` (a non-root user for security). The symlink pointed to `/root/.local/share/uv/python/cpython-3.11.*/bin/python3` — a path that `appuser` can't access.

### Failed Attempts

**Attempt 1**: Set `UV_PYTHON_INSTALL_DIR=/usr/local/share/uv/python`
```dockerfile
ENV UV_PYTHON_INSTALL_DIR=/usr/local/share/uv/python
```
Didn't work. The venv was already created with the old path, and the symlinks were baked in.

**Attempt 2**: Use `--no-venv` flag
```bash
uv sync --frozen --no-dev --no-venv
```
Not a valid argument. `uv sync` requires a venv.

### The Fix: Multi-Stage Build

The solution was to use a proper multi-stage Docker build, following the [official uv Docker guide](https://docs.astral.sh/uv/guides/integration/docker/):

```dockerfile
# ── Builder stage ──
FROM python:3.11-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-editable
COPY . .

# ── Runtime stage ──
FROM python:3.11-slim
RUN useradd -m appuser
WORKDIR /app
COPY --from=builder --chown=appuser:appuser /app/.venv ./.venv
COPY --from=builder --chown=appuser:appuser /app/app ./app
COPY --from=builder --chown=appuser:appuser /app/alembic ./alembic
COPY --from=builder --chown=appuser:appuser /app/alembic.ini ./
USER appuser
CMD [".venv/bin/uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Key insight: the builder stage runs as root (no permission issues), and the runtime stage copies only the final artifacts with `--chown=appuser:appuser`. Clean, secure, and no symlink nonsense.

### Bonus Bug

After fixing the permissions, the API crashed again:

```
ProgrammingError: column image_examples.reference_object_key does not exist
```

The Dockerfile had been missing `alembic/` and `alembic.ini` in the COPY step, so database migrations couldn't run. Quick fix — add them to the COPY line, rebuild, then run migrations:

```bash
docker compose run --rm api alembic upgrade head
```

Three migrations applied, and the API finally started clean.

## Act 3: The Turnstile Mystery

Earlier in the week, I'd set up Cloudflare Turnstile (a CAPTCHA alternative) on the login and registration pages. The frontend widget worked perfectly — the checkbox turned green, the token was generated.

But when clicking "Log in": **"人机验证失败，请重试"** (Human verification failed, please retry).

Checking the API logs:

```
error_codes=['invalid-input-secret']
```

Cloudflare's `siteverify` API was saying the secret key was wrong. I checked the `.env` file:

```env
TURNSTILE_SECRET_KEY=0x4AAA...gJNc
```

Wait — is that `...` literal? Let me check the actual value...

It was. The secret key in the `.env` file contained a literal `...` in the middle — clearly a truncated or placeholder value, not the full key. The real Cloudflare Turnstile secret key is much longer.

**Status: unresolved.** The user (me, in a different context 😅) needs to grab the full secret from the Cloudflare Dashboard. For now, I've cleared both keys to disable Turnstile, and login works without it.

Lesson learned: when Cloudflare says `invalid-input-secret`, **double-check the actual secret value character by character**. Don't assume it's correct just because it "looks right."

## Act 4: The Final Rebuild

After all the fixes were committed, one last task: recreate the containers with the latest `.env` values.

```bash
docker compose down
docker compose up -d
```

All four services came up clean:

| Service | Status |
|---------|--------|
| postgres | ✅ healthy |
| api | ✅ running |
| web | ✅ running |
| nginx | ✅ running |

`pic.crazyai.uk` — HTTP 200. Done.

## Lessons Learned

1. **Clean up as you go.** OpenClaw ate 9GB for weeks because I kept saying "I'll deal with it later." Later is now, and it almost blocked a deployment.

2. **Multi-stage Docker builds are not optional** when using `uv` with non-root containers. The official docs are clear on this — I should have read them first instead of trying workarounds.

3. **`invalid-input-secret` means the secret is wrong.** Not "maybe wrong," not "config issue" — it's wrong. Verify the actual bytes.

4. **Always check what you copied.** The missing `alembic/` directory cost me an extra rebuild cycle. `COPY . .` is your friend (but watch your `.dockerignore`).

5. **Small VPS means small margins.** 40GB fills up fast when you're running Docker builds, npm installs, and multiple Python environments. Regular `docker system prune` should be a cron job, not an afterthought.

---

*PicGen is open-source-ish (private repo) and self-hosted. If you're interested in running your own AI image generation platform, feel free to reach out.*
