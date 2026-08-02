---
id: url-batch-import-recipe-quick-start
title: URL Batch Import Recipe — Quick Start
status: active
created: 2026-06-13T03:12:39.688Z
updated: 2026-08-01T06:28:27.547Z
tags:
  - recipes
  - url-import
  - open-brain
---

# URL Batch Import Recipe — Quick Start

## Status (2026-08-01)
PR #339 (upstream contribution to `NateBJones-Projects/OB1`) was **withdrawn/closed** — recipe remains in active personal use on `kae36/personal`; only the upstream contribution effort ended. See checkpoint `checkpoint_3c0971ed` for full context.

The broader "adopting upstream OB1 tools + Open Brain redeploy" thread that was tracked here has been split out into its own brief: **[[bring-my-open-brain-up-to-date]]**. This brief now covers just the URL batch import recipe itself.

## What It Does
Fetches a list of URLs (one per line in `.txt` or rows in `.csv`), extracts readable text, summarizes with LLM via OpenRouter, and stores each as a searchable thought in Open Brain with deduplication via `sync-log.json`.

## Known unresolved (local use)
14 queued fetch failures (mostly Dotdash Meredith 403 bot-block cluster) need a headless-browser/logged-in-session approach to crack.
