---
id: bring-my-open-brain-up-to-date
title: Bring My Open Brain Up To Date
status: active
created: 2026-08-01T06:28:46.561Z
updated: 2026-09-20T16:57:17.255Z
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
Adopt useful new upstream OB1 capabilities into my personal Open Brain (Supabase project `dpcybjkruqrbdlyvfeqb`, "kae36's Org"). Strategy since 2026-09-13: fully sync this fork with upstream first, reconcile security across everything that syncs in, then pick new capabilities one at a time.

## Status as of 2026-09-20: WAITING ON UPSTREAM

**Decision 2026-09-20 (user): pause the adoption queue and wait for upstream to merge the fix PR (#490) before touching `enhanced-mcp` or anything else that captures thoughts.** No local patch for now. Stock `open-brain` connector is the working capture/search path (verified 2026-09-20: thought_stats 799, list, semantic search).

**`open-brain-enhanced` registration REMOVED from `~/.claude.json` on 2026-09-20** (`claude mcp remove open-brain-enhanced -s user`; it was user-scope only, never registered on another machine). Its tools disappear from a session after restart. The deployed `enhanced-mcp` edge function is still live at its Supabase URL and answers to the project-wide `MCP_ACCESS_KEY`; deleting it is the user's call at the console. No database rollback is needed: enhanced-mcp itself changed no DB objects (per the 2026-09-19 record: function + secrets only; the `enhanced-thoughts` schema it depends on was already live and is compatible with the stock connector, see below).

**Upstream sync: done, twice.** ~350 commits 2026-09-13, 59 commits 2026-09-18. Upstream `main` HEAD was `238df6c` (2026-09-10) as of 2026-09-20, already merged here. Every queued capability exists in the repo tree; none is in use against the live DB.

### Stock connector vs the enhanced-thoughts DB changes (checked 2026-09-20)
Compatible. Stock `server/index.ts` `capture_thought` calls `upsert_thought` with `{metadata: {..., source: "mcp"}}` and reads `upsertResult.id` then writes the embedding itself, which matches the `{id, fingerprint}` contract the enhanced-thoughts RPC keeps. Stock search calls `match_thoughts` with `filter: {}`; list/stats read the table directly. Verified live today: stats, list, semantic search. Not tested live today: stock `capture_thought` (evidence only: thoughts captured 2026-09-17 with stock-style metadata after enhanced-thoughts was live). The problems below are all in enhanced-mcp/rest-api client code, not in the DB.

### Why we are waiting (upstream drift vs our live DB)
Live `thoughts.id` is UUID and the shipped `upsert_thought` returns `{id, fingerprint}`, ignores top-level `p_payload` fields (so detected sensitivity tier, importance/quality and the embedding are not persisted), and both search RPCs treat `p_filter` as `metadata @> filter`. Verified 2026-09-20 against the live enhanced connector: classification works and the restricted-content write gate works, but capture returns a false error after storing the row, and both search modes return nothing. `rest-api` and `smart-ingest` share the same helpers/RPC, so they hit the same capture problem plus digit-only id routes (`rest-api`).
- Known upstream issue: **#379** (UUID/bigint mismatch on `upsert_thought` return), also #456, #242.
- **PR #490** (TomKeyser, `integrations/enhanced-mcp/index.ts` only, +56/-25, open since 2026-09-02, mergeable, review required): fixes zero-row search (both modes), false capture error + persists the embedding via follow-up update, UUID `get_thought`. Edge-function redeploy only, no DB change. Does NOT fix `update_thought`/`related_thoughts` integer ids (PR #506 drops `update_thought`) or persist the sensitivity tier/importance/quality (stay `standard`/50/70).
- **PR #363** (alanshurafa, schema, +472/-56): server-side date/tier filters in `search_thoughts_text` but does not fix capture (still `{id, fingerprint}`, no embedding insert). Rewrites a live function; not recommended.
- Other open PRs: #413, #506, #430 (enhanced-mcp), #504/#429/#367/#473 (smart-ingest UUID). None merged. No PR for `rest-api` was found.

### How to re-check upstream (read-only)
```
gh pr view 490 --repo NateBJones-Projects/OB1 --json state,mergedAt
gh pr view 506 --repo NateBJones-Projects/OB1 --json state,mergedAt
gh api repos/NateBJones-Projects/OB1/commits/HEAD --jq '.sha[0:7] + " " + .commit.committer.date'
```
If `HEAD` is no longer `238df6c`, do the third upstream sync. To bring enhanced-mcp back: redeploy the fixed function at the console, re-register with `claude mcp add` (user scope, HTTP, `x-brain-key` header), then re-run the verification (capture returns success, row has embedding, text + semantic search return hits, `get_thought` by UUID). `gh`/`git` need the sandbox disabled on this machine (sandbox setup fails with an `apply-seccomp` error); `gh` reads only.

## Security (corrected 2026-09-19)
- Aug 5 remediation (RLS + revokes on `crm-person-tiers`, `security_invoker=true` on `brain-health-monitoring` ops views) intact.
- 2026-09-13 fixed the enhanced-thoughts SECURITY DEFINER exposure (`brain_stats_aggregate`, `get_thought_connections`) INCOMPLETELY (revoked from `anon, authenticated`, not `PUBLIC`). Actually fixed and verified live 2026-09-19 (`proacl` = owner + `service_role` only).
- **Rule:** revoke from `PUBLIC, anon, authenticated`; verify with `has_function_privilege()` and raw `proacl::text`, never a `pg_roles` join. Repo schema REVOKEs naming only `anon` have the same gap (PR #363's new function would too).
- Any `search_path` pin on a function using extension types must include `extensions` (pinning `match_thoughts` broke pgvector search until fixed).
- Landmine catalog `.planning/supabase-security/landmine-catalog-2026-09-13.md` (with 2026-09-19 addendum): 12 SECURITY DEFINER functions, 2 not-yet-deployed landmines (`editorial-policy` `get_recent_audit_reports`; `readwise-books` two anon grants and no RLS). Read it before deploying any schema.
- Family-calendar checked live 2026-09-19: not exposed (RLS on, anon has no table privileges). Repo schema file lacks RLS statements but live has it on, so the repo file is not the source of truth for live state.
- Lower priority: several SECURITY INVOKER functions (`match_thoughts`, `upsert_thought`, `search_thoughts_text`, `crm_person_tiers`, `brain_stats_daily*`) are still anon-executable; low risk while anon has no table privileges on `thoughts`.

## Housekeeping
- Machine-local ignores live in `.git/info/exclude` (per clone); `.gitignore` matches upstream. Commit `d8a5f6a`, pushed.
- Todo-list workflow already works natively via the thoughts schema ("Work task: ..." / "Home task: ...").
- The 721 thoughts with a null `type` column already carry types in stock metadata (news 401, analysis 144, ...); `backfill_thought_types()` only copies the 8 canonical types, so it is moot unless something needs the typed column.

## Open items
1. Wait for upstream #490 (and preferably #506) to merge, then sync and re-verify (see above). No other action needed until then.
2. Optional: delete the deployed `enhanced-mcp` edge function at the console if it should not be reachable at all.
3. `contrib/kae36/url-batch-import` still has 6 unmerged commits (unrelated URL-import recipe).
4. `.memories/` checkpoints from 2026-09-20 are uncommitted.

## Key files
- `.planning/supabase-security/` (gitignored, local): fix scripts, `check-live-exposure-2026-09-19.sql`, `fix-definer-public-grant-2026-09-19.sql`, landmine catalog.
- Deploy playbook and machine notes from the original brief still apply (M7GASG5 console, Claude Desktop launch flag, dashboard SQL editor shows only the last statement's result, no local psql). User does deploys and secrets at the console.

