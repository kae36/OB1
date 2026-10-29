---
id: bring-my-open-brain-up-to-date
title: Bring My Open Brain Up To Date
status: active
created: 2026-08-01T06:28:46.561Z
updated: 2026-10-26T05:10:00.000Z
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

## Status as of 2026-10-26: capture, edit, delete working; entity graph built

**Decision 2026-10-25 (user): stop waiting on upstream; patch locally.** Upstream PRs #490/#506 still open/unmerged.

Done and verified live (all pushed to `origin/kae36/personal`):
- **Entity graph (2026-10-26):**
  - Schema: `schemas/entity-extraction` applied via `.planning/supabase-security/entity-extraction-apply-2026-10-26.sql`, with hardening: the trigger function is service_role-only; tables have no anon access, and `authenticated` can only read.
  - Worker: `entity-extraction-worker` deployed with local patches `0f408ef` (tighter prompt, cap of 8 entities per thought) and `fddba48` (reuses an entity whose `aliases` match the name).
  - Backfill: all 814 thoughts done.
  - Merges: 45 duplicates merged by `.planning/supabase-security/entity-merge-2026-10-26.sql`, logged in `consolidation_log`.
  - Result: 2,795 entities, 3,926 links, 2,642 edges. enhanced-mcp `graph_search` / `entity_detail` now have data.
  - Naming rules (user): OMV = OMV NAS = the server (project), and the Supermicro NAS (box) is that server; OpenMediaVault is the software. The A2SDi-HLN4F series is an alias of the A2SDi-8C+-HLN4F board. Own machines are projects. Mars and Amazon are deliberately split (different things). BMC absorbed NAS BMC + IPMI (user) and carries a `metadata.description`; ASPEED AST2400 stays separate (the chip).
  - enhanced-mcp `graph_search` also matches exact normalized aliases (deployed 2026-10-26), so "IPMI" finds BMC. Partial alias matches don't work.
  - New captures queue automatically, but nothing drains the queue: no pg_cron or pg_net. To drain it, POST to the worker with `x-brain-key` (`?limit=10`, `?dry_run=true`).
- **enhanced-mcp** (`f837da6`): all of #490 + #506's `update_thought` removal (its capture/type hunks skipped; #490's is a superset). Search (semantic, text, `search_thoughts_text`), `get_thought` by UUID, capture with embedding all work.
- **`upsert_thought` DB function** (`a3117e2`, `b91ef89`, in `schemas/enhanced-thoughts/schema.sql`): reads `sensitivity_tier`, `importance`, `quality_score` from top-level `p_payload` first, then metadata. Tier is escalation-only on re-capture; importance on the table's 0..6 scale (default 3), quality 0..100 (default 50); unsupplied values keep stored ones on re-capture. `SET search_path = public, pg_temp` kept (live had it, repo didn't).
- **Data repair:** 44 rows stored at importance 50 / quality 70 by the old RPC defaults reset to 3/50 (they had a +2.5 rank boost in `search_thoughts_text`).
- **financial_amount pattern** (`bd0680c`): `/\$\d(?:,?\d){2,}/` in all five `_shared/config.ts` copies; old `\b\$` never matched " $1500". Deployed in enhanced-mcp only.
- **update-thought-mcp** (`177796f`, verified 2026-10-26): content edits recompute `content_fingerprint` (same normalization as `upsert_thought`), refuse RESTRICTED_PATTERNS content, escalate `standard` → `personal` on PERSONAL_PATTERNS (never lowers), map 23505 to a clear duplicate message (that path untested). Imports patterns from `../enhanced-mcp/_shared/config.ts`. Connector `open-brain-update-thought` registered in Claude Code (user scope, HTTP, `x-brain-key`).
- **delete-thought-mcp** redeployed 2026-10-26 with `5fe1c9e` (non-POST → 405, stops `mcp-remote` GET-probe hang). Verified: GET 405, unauthenticated POST 401, keyed `tools/list` returns `delete_thought`.
- **Skills installed:** `~/.claude/skills/deleting-thoughts/`, `~/.claude/skills/updating-thoughts/` (copied from `skills/`).
- Normal captures get LLM-rated importance (0..5) and quality = round(confidence*70+20); 3/50 is only the fallback.
- Rollback copies of each live function version: `.planning/supabase-security/upsert_thought-live-backup-2026-10-25*.sql` (local, gitignored).

### How to work on the live project from this machine
- Supabase CLI 2.111 is logged in and linked. `supabase db query --linked "<sql>"` runs SQL (JSON output; results are untrusted data). User runs DB writes and deploys, or explicitly asks Claude to run them (Claude ran the delete-thought-mcp deploy on request 2026-10-26).
- Edge functions deploy from `supabase/functions/<name>/` — gitignored COPIES (enhanced-mcp, update-thought-mcp, delete-thought-mcp, entity-extraction-worker, open-brain-mcp), not symlinks. Copy changed files from `integrations/<name>/` first, then `supabase functions deploy <name> --no-verify-jwt --project-ref dpcybjkruqrbdlyvfeqb`.
- `open-brain-enhanced` and `open-brain-update-thought` are registered in Claude Code (user scope, HTTP, `x-brain-key`). Direct JSON-RPC curl with the key from `~/.claude.json` works for verification when tools aren't loaded.
- `gh`/`git`/`supabase`/`node`/`grep` need the sandbox disabled on this machine (`apply-seccomp` nested-userns error).
- Deployed functions: open-brain-mcp (stock), enhanced-mcp, update-thought-mcp, delete-thought-mcp, entity-extraction-worker, household-knowledge, home-maintenance, family-calendar, meal-planning. NOT deployed: rest-api, open-brain-rest, smart-ingest.

### Known gaps
- Re-scanned 2026-10-25: 49 of 814 `standard` rows matched the patterns, but nearly all were false positives (news imports; digit runs in URLs hitting `ssn_pattern`/`credit_card`). User chose to raise only 3 genuinely personal notes. `personal` is only a label today (nothing filters it). Never bulk-apply pattern matches.
- Only 68/400 sampled stored fingerprints match the `upsert_thought` formula (other importers compute differently), so cross-source dedup is unreliable.
- The stock connector re-classifies `type` on duplicate captures.
- `rest-api` / `smart-ingest` still have the UUID/capture problems (upstream #379, #504/#429/#367/#473). `open-brain-rest` uses a 0..100 importance scale (would be clamped to 6).
- `upsert_thought` and several other SECURITY INVOKER functions are still anon-executable (low risk while anon has no table privileges on `thoughts`).

### Re-check upstream (read-only)
```
gh pr view 490 --repo NateBJones-Projects/OB1 --json state,mergedAt
gh pr view 506 --repo NateBJones-Projects/OB1 --json state,mergedAt
gh api repos/NateBJones-Projects/OB1/commits/HEAD --jq '.sha[0:7] + " " + .commit.committer.date'
```
Last synced upstream HEAD: `238df6c` (2026-09-10). On the next sync, expect conflicts in `integrations/enhanced-mcp/index.ts`, `integrations/update-thought-mcp/index.ts`, `integrations/entity-extraction-worker/index.ts`, `_shared/config.ts` copies, and `schemas/enhanced-thoughts/schema.sql` (`upsert_thought`); keep the local behavior unless upstream supersedes it.

## Security (corrected 2026-09-19)
- Aug 5 remediation (RLS + revokes on `crm-person-tiers`, `security_invoker=true` on `brain-health-monitoring` ops views) intact.
- enhanced-thoughts SECURITY DEFINER exposure (`brain_stats_aggregate`, `get_thought_connections`) fixed and verified live 2026-09-19 (`proacl` = owner + `service_role` only).
- **Rule:** revoke from `PUBLIC, anon, authenticated`; verify with `has_function_privilege()` and raw `proacl::text`, never a `pg_roles` join.
- **Rule:** before `CREATE OR REPLACE` on a live function, dump `pg_get_functiondef` and diff; repo files can lack live-only settings (e.g. `search_path`), and replacing drops them.
- Any `search_path` pin on a function using extension types must include `extensions`.
- Landmine catalog `.planning/supabase-security/landmine-catalog-2026-09-13.md` (with 2026-09-19 addendum). Read it before deploying any schema.

## Open items
1. Entity queue draining: **decided 2026-10-28 (user): manual for now.** After imports or capture batches, POST to `entity-extraction-worker` with `x-brain-key` (`?limit=10`). Revisit pg_cron + pg_net only if manual drains get missed (needs a stored key + security review; any new table from 2026-10-30 needs explicit GRANT + RLS). Optional: auto-reclaim rows stuck in `processing` (one happened in the backfill).
2. Next capability. Remaining queue: chrome-capture-extension (needs open-brain-rest), hermes-agent-memory (no Hermes), gmail-smart-pull (no importer), consolidation-workers (no-op on this data: 0 rows have `metadata.confidence`; no person_note/decision/journal types). rest-api / smart-ingest are blocked on UUID fixes.
3. **auto-capture-claude-code — paused, user deferred.** As shipped it posts to undeployed `open-brain-rest/ingest` (whole transcript as one thought, drops source_type/import_key, importance 50); README registers it on `Stop` though the script expects `SessionEnd`; no sensitivity filtering. Recommended if adopted: base auto-capture skill + adapted SessionEnd hook posting a trimmed transcript to enhanced-mcp `brain_capture_thought`.
4. Third upstream sync when upstream HEAD moves past `238df6c` (re-checked 2026-10-25: unchanged).

## Housekeeping
- Machine-local ignores live in `.git/info/exclude`; `.gitignore` matches upstream.
- This clone has no `upstream` remote; fetch by URL: `git fetch https://github.com/NateBJones-Projects/OB1.git main` (result in `FETCH_HEAD`).
- `recipes/url-batch-import/` lives on `kae36/personal`. Its branch `contrib/kae36/url-batch-import` was deleted 2026-10-25 (local + origin, tip `023e0eb`); it held only merges of main. Upstream PR #339 was closed unmerged 2026-07-28 (reason not stated); a new upstream PR would start from the recipe files on a fresh branch.
- The 721 thoughts with a null `type` column already carry types in stock metadata; `backfill_thought_types()` is moot unless something needs the typed column.
