---
id: bring-my-open-brain-up-to-date
title: Bring My Open Brain Up To Date
status: active
created: 2026-08-01T06:28:46.561Z
updated: 2026-09-19T06:04:43.383Z
tags:
  - open-brain
  - upstream-adoption
  - supabase
  - m7gasg5
  - mcp-connectors
  - security
---

# Bring My Open Brain Up To Date

## Goal
Adopt useful new upstream OB1 capabilities into my personal Open Brain instance (Supabase project `dpcybjkruqrbdlyvfeqb`, "kae36's Org"). Strategy since 2026-09-13: fully sync this fork with upstream first, reconcile security across everything that syncs in, then pick new capabilities one at a time.

## Status as of 2026-09-19

**Upstream sync: done, twice.** First sync (~350 commits) 2026-09-13, second (59 commits) 2026-09-18. Every capability in the adoption queue (`enhanced-mcp`, `hermes-agent-memory`, `rest-api`, `smart-ingest`, `chrome-capture-extension`, `consolidation-workers`, `gmail-smart-pull`, `auto-capture-claude-code`) exists in the repo tree, current with upstream. **`enhanced-mcp` is now deployed and connected (2026-09-19); the rest are not deployed to the live DB yet.**

**Security (corrected 2026-09-19):**
- Aug 5 remediation (RLS + revokes on `crm-person-tiers`, `security_invoker=true` on `brain-health-monitoring` ops views) intact.
- 2026-09-13 fixed the enhanced-thoughts SECURITY DEFINER exposure (`brain_stats_aggregate`, `get_thought_connections`), but INCOMPLETELY: it revoked from `anon, authenticated` and not `PUBLIC`, so anon could still execute both, and the 09-13 verification (a `pg_roles` join that drops PUBLIC) could not see it. **Actually fixed and verified live 2026-09-19** (`proacl` = owner + `service_role` only; `has_function_privilege` false for anon and authenticated). The "verified live" claim in the 09-13 records was wrong.
- **Rule:** revoke from `PUBLIC, anon, authenticated`; verify with `has_function_privilege()` and raw `proacl::text`, never a `pg_roles` join. Repo schema REVOKEs naming only `anon` have the same gap.
- 09-13 also pinned `search_path` on 6 functions; pinning `match_thoughts` broke pgvector search until `extensions` was added back. Any search_path pin on a function using extension types must include `extensions`.
- Landmine catalog `.planning/supabase-security/landmine-catalog-2026-09-13.md` (with 2026-09-19 addendum + correction): still 12 SECURITY DEFINER functions, 2 not-yet-deployed landmines (`editorial-policy` `get_recent_audit_reports`; `readwise-books` two anon grants and no RLS). The second sync's 3 new SQL files are clean.
- **Family-calendar checked live 2026-09-19: not exposed.** Every public table has RLS on and `anon` has no table privileges (13 tables incl. family_members, activities, important_dates; nothing EXPOSED). The repo schema file lacks RLS statements but the live DB has it on (cause unverified, possibly the project's auto-enable-RLS setting), so the repo file is not the source of truth for live state. `recipes/repo-learning-coach` has the same file-level gap but is not deployed.
- Lower priority: several SECURITY INVOKER functions (`match_thoughts`, `upsert_thought`, `search_thoughts_text`, `crm_person_tiers`, `brain_stats_daily*`) are still anon-executable; low risk while anon has no table privileges on `thoughts`.

**Housekeeping done 2026-09-19:** machine-local ignores moved from tracked `.gitignore` to `.git/info/exclude` (per-clone, so other clones need the same lines); `.gitignore` now matches upstream. Commit `d8a5f6a`, pushed.

**Todo-list workflow** already works natively via the thoughts schema ("Work task: ..." / "Home task: ...").

## enhanced-mcp: deployed and connected (2026-09-19)
A second remote MCP connector (Supabase Edge Function, `functions/v1/enhanced-mcp`) beside the stock one: 13 tools, `brain_`-prefixed versions of the 4 stock tools with extra filters, auto-classification, sensitivity detection and fingerprint dedup. Registered as the user-scoped HTTP server `open-brain-enhanced` in `~/.claude.json` with an `x-brain-key` header. Prerequisite `enhanced-thoughts` is already live. Needs `MCP_ACCESS_KEY` and `OPENROUTER_API_KEY` as function secrets: the user does deploy and secrets at the console. Runs as service_role, so the function is the sensitivity-filter boundary.

**Verified:** connects; read-only `brain_thought_stats` returned 799 thoughts (721 with `type: null`, 50 reference, 14 observation, 12 task, 2 idea).
**Not verified:** write path (`brain_capture_thought`), auto-classification (needs `OPENROUTER_API_KEY`), and that the sensitivity filter actually hides restricted thoughts from the read tools.

**Connection failure lesson:** the first connect failed because the deployed `MCP_ACCESS_KEY` differed from the key in the client config. The function returned 401, and Claude Code showed that as a confusing "Dynamic Client Registration rejected (HTTP 404)". If it recurs, probe the endpoint directly first. `supabase secrets list` shows only SHA-256 digests, never values, so a digest cannot be used as the key. `MCP_ACCESS_KEY` is project-wide, so do not rotate it without checking what else reads it.

Later in the queue: `rest-api`, then `chrome-capture-extension` / `auto-capture-claude-code` (which depend on it); `auto-capture-claude-code` overlaps with Goldfish and sends full transcripts, so likely skip.

## Open items
1. Verify the enhanced-mcp write path, auto-classification and the sensitivity filter (see Not verified above), then decide whether to backfill classification for the 721 untyped thoughts.
2. Confirm the stock `open-brain` connector still works with whatever `MCP_ACCESS_KEY` is now live.
3. Pick the next capability (`rest-api` is next in the queue); check the landmine catalog first.
4. `contrib/kae36/url-batch-import` still has 6 unmerged commits (unrelated URL-import recipe).

## Key files
- `.planning/supabase-security/` (gitignored, local): fix scripts, `check-live-exposure-2026-09-19.sql`, `fix-definer-public-grant-2026-09-19.sql`, landmine catalog.
- Deploy playbook and machine notes from the original brief still apply (M7GASG5 console, Claude Desktop launch flag, dashboard SQL editor shows only the last statement's result, no local psql).
