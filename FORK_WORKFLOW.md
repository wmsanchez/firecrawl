# Fork Workflow Guide

This repository is a personal fork of [Firecrawl](https://github.com/mendableai/firecrawl) by Mendable. This document explains how our repo is set up and how to keep it in sync with upstream while preserving our customizations.

## Repository Setup

| Remote     | URL                                          | Purpose                              |
|------------|----------------------------------------------|--------------------------------------|
| `origin`   | https://github.com/wmsanchez/firecrawl.git   | Our fork - push changes here         |
| `upstream` | https://github.com/firecrawl/firecrawl.git   | Original Mendable repo - pull updates |

### Verify remotes
```bash
git remote -v
```

---

## Our Customizations

Keep track of files we've modified from the original:

| File | Type | Description |
|------|------|-------------|
| `.gitignore` | Modified | Added local config files to ignore list |
| `docker-compose.yaml` | Ignored | Local Docker configuration (not tracked) |
| `litellm-config.yaml` | Ignored | LiteLLM Proxy config for Bedrock routing (not tracked) |
| `searxng/settings.yml` | Ignored | Local SearXNG settings (not tracked) |
| `FORK_WORKFLOW.md` | New | This documentation file |

### Ignored Local Files

These files are in `.gitignore` and won't be committed. They contain our local-only configurations:

- `docker-compose.yaml` - Use `docker-compose.example.yaml` from upstream as reference
- `litellm-config.yaml` - LiteLLM Proxy model routing (Bedrock)
- `searxng/settings.yml` - Local search engine settings

> **Tip:** When making new customizations, add them to this list to track what may need attention during merges.

---

## Syncing with Upstream

### Option 1: Merge (Recommended for most cases)

Preserves complete history and is safer for shared branches.

```bash
# 1. Fetch latest changes from Mendable
git fetch upstream

# 2. Make sure you're on main
git checkout main

# 3. Merge upstream changes
git merge upstream/main

# 4. Resolve any conflicts (see Conflict Resolution below)

# 5. Push to our fork
git push origin main
```

### Option 2: Rebase (Cleaner history, use with caution)

Replays our commits on top of upstream. Only use if you haven't shared your branch.

```bash
git fetch upstream
git checkout main
git rebase upstream/main

# If conflicts occur, resolve them, then:
git rebase --continue

# Force push (required after rebase)
git push origin main --force-with-lease
```

---

## Conflict Resolution

When upstream changes conflict with our customizations:

1. **Git will mark conflicts** in the affected files
2. **Open the conflicted files** and look for conflict markers:
   ```
   <<<<<<< HEAD
   (our changes)
   =======
   (upstream changes)
   >>>>>>> upstream/main
   ```
3. **Edit the file** to keep what you need from both versions
4. **Mark as resolved:**
   ```bash
   git add <resolved-file>
   git commit  # or `git rebase --continue` if rebasing
   ```

---

## Best Practices to Minimize Tech Debt

### 1. Keep customizations isolated

When possible, use **override files** instead of modifying originals:

- `docker-compose.override.yml` → Overrides `docker-compose.yaml` automatically
- `.env.local` → Local environment variables
- Create separate config files that import/extend originals

### 2. Use a dedicated branch for heavy customizations

```bash
# Create a customizations branch
git checkout -b customizations

# Keep main clean and synced with upstream
git checkout main
git merge upstream/main
git push origin main

# Periodically rebase customizations onto updated main
git checkout customizations
git rebase main
```

### 3. Document every modification

Update the "Our Customizations" section above whenever you modify upstream files.

### 4. Consider upstream-friendly changes

If your change could benefit everyone, consider contributing it back:
1. Create a feature branch from upstream/main
2. Make the change
3. Submit a PR to the original repo

### 5. Review upstream changes before merging

```bash
# See what's new in upstream
git fetch upstream
git log main..upstream/main --oneline

# See the actual changes
git diff main..upstream/main
```

---

## LLM Setup — Amazon Bedrock via LiteLLM Proxy

Firecrawl needs an LLM for features like extraction, smart scraping, and deep research. Instead of modifying Firecrawl's code (which hardcodes OpenAI as the provider), we run a **LiteLLM Proxy** container that translates OpenAI-compatible API calls to Amazon Bedrock.

### How it works

```
Firecrawl (thinks it's talking to OpenAI)
  → OPENAI_BASE_URL = http://litellm:4000/v1
    → LiteLLM Proxy (OpenAI-compatible gateway)
      → Amazon Bedrock (Claude, Titan embeddings)
```

### Configuration files

| File | Purpose |
|------|---------|
| `litellm-config.yaml` | Model routing — maps OpenAI model names to Bedrock models |
| `.env` | AWS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION_NAME`) |

### Model mapping

| Firecrawl requests | LiteLLM routes to |
|-------------------|-------------------|
| `gpt-4o` | `bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0` |
| `gpt-4o-mini` | `bedrock/anthropic.claude-3-haiku-20240307-v1:0` |
| `o3-mini` | `bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0` |
| `text-embedding-3-small` | `bedrock/amazon.titan-embed-text-v2:0` |

### Updating models

Edit `litellm-config.yaml` and restart:
```bash
docker compose restart litellm
```

### Testing the LLM connection

```bash
# Check LiteLLM health
curl http://localhost:4000/health/liveliness

# Test a chat completion through LiteLLM
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-litellm" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Say hello"}]}'
```

---

## Keeping Up-to-Date — Pre-built Images

We use pre-built Docker images from `ghcr.io/firecrawl/` instead of building from source. To update:

```bash
# Pull latest images and restart
docker compose pull && docker compose up -d
```

Since we don't modify any application code, this is safe and avoids build times.

---

## Quick Reference

| Task | Command |
|------|---------|
| Check current remotes | `git remote -v` |
| Fetch upstream updates | `git fetch upstream` |
| See new upstream commits | `git log main..upstream/main --oneline` |
| Merge upstream | `git merge upstream/main` |
| Push to our fork | `git push origin main` |
| Abort a bad merge | `git merge --abort` |
| Abort a bad rebase | `git rebase --abort` |
| Update Docker images | `docker compose pull && docker compose up -d` |
| Restart LiteLLM after config change | `docker compose restart litellm` |
| Test LLM connection | `curl http://localhost:4000/health/liveliness` |

---

## Recovery Commands

If something goes wrong:

```bash
# Undo last commit (keep changes staged)
git reset --soft HEAD~1

# Undo last commit (discard changes) ⚠️ DESTRUCTIVE
git reset --hard HEAD~1

# Restore a file to upstream version
git checkout upstream/main -- path/to/file

# View reflog to find lost commits
git reflog
```

---

*Last updated: March 2026*
