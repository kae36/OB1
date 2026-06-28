---
id: url-batch-import-recipe-quick-start
title: URL Batch Import Recipe — Quick Start
status: active
created: 2026-06-13T03:12:39.688Z
updated: 2026-06-13T03:12:39.688Z
tags:
  - recipes
  - url-import
  - open-brain
---

## What It Does
Fetches a list of URLs (one per line in `.txt` or rows in `.csv`), extracts readable text, summarizes with LLM via OpenRouter, and stores each as a searchable thought in Open Brain with deduplication via `sync-log.json`.

## Prerequisites
- Deno runtime installed (`deno --version`)
- `.env` file in `recipes/url-batch-import/` with:
  ```bash
  export SUPABASE_URL=https://YOUR_REF.supabase.co
  export SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
  export OPENROUTER_API_KEY=sk-or-v1-your-key
  ```
- Input file: `.txt` (one URL per line) or `.csv` (with `url` column)

## Basic Command
```bash
cd recipes/url-batch-import

# Set environment variables in this session
source .env  # (or manually: export VAR=value for each)

# Run the import
deno run --allow-net --allow-read --allow-write --allow-env import-urls.ts --input=path/to/urls.txt
```

## Common Options
- `--dry-run` — Preview without writing to Open Brain
- `--limit=N` — Process only first N URLs
- `--offset=N --limit=M` — Process URLs N through N+M (for batching large files)
- `--ingest-endpoint` — Use custom edge function instead of direct Supabase insert

## Deduplication
- `sync-log.json` tracks processed URLs
- Re-running the same file only imports NEW URLs
- Safe for growing lists and cross-machine resumption
- Delete `sync-log.json` to re-import all URLs

## Troubleshooting
- **Env vars not found:** Ensure `source .env` runs BEFORE the deno command in same shell session
- **Fetch failures:** Check `failures.log` for details (timeouts, 429, unsupported MIME type, etc.)
- **No thoughts appear:** Verify `SUPABASE_SERVICE_ROLE_KEY` is the service role (not anon key)
