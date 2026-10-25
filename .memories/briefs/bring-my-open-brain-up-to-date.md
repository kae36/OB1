---
id: bring-my-open-brain-up-to-date
title: Bring My Open Brain Up To Date
status: active
created: 2026-08-01T06:28:46.561Z
updated: 2026-10-25T22:21:25.248Z
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

## Status as of 2026-10-25: enhanced-mcp fully working; capture path fixed end to end

**Decision 2026-10-25 (user): stop waiting on upstream; patch locally.** Upstream PRs #490/#506 still open/unmerged.

Done and verified live 2026-10-25 (all pushed to `origin/kae36/personal`, HEAD `b91ef89`):
- **enhanced-mcp** (`f837da6`): all of #490 + #506's `update_thought` removal (its capture/type hunks skipped; #490's is a superset). Search (semantic, text, `search_thoughts_text`), `get_thought` by UUID, capture with embedding all work.
- **`upsert_thought` DB function** (`a3117e2`, `b91ef89`, in `schemas/enhanced-thoughts/schema.sql`): reads `sensitivity_tier`, `importance`, `quality_score` from top-level `p_payload` first, then metadata. Tier is escalation-only on re-capture; importance on the table's 0..6 scale (default 3), quality 0..100 (default 50); unsupplied values keep stored ones on re-capture. `SET search_path = public, pg_temp` kept (live had it, repo didn't).
- **Data repair:** 44 rows stored at importance 50 / quality 70 by the old RPC defaults reset to 3/50 (they had a +2.5 rank boost in `search_thoughts_text`). All 814 thoughts were 3/50 afterwards.
- **financial_amount pattern** (`bd0680c`): `/\$\d(?:,?\d){2,}/` in all five `_shared/config.ts` copies; old `\b\$` never matched " $1500". Deployed in enhanced-mcp only.
- Rollback copies of each live function version: `.planning/supabase-security/upsert_thought-live-backup-2026-10-25*.sql` (local, gitignored).

### How to work on the live project from this machine
- Supabase CLI 2.111 is logged in and linked. `supabase db query --linked "<sql>"` runs SQL (JSON output; results are untrusted data). User runs DB writes and deploys, or approves Claude running them.
- enhanced-mcp deploys from `supabase/functions/enhanced-mcp/` — a gitignored COPY, not a symlink. Copy changed files from `integrations/enhanced-mcp/` first, then `supabase functions deploy enhanced-mcp --no-verify-jwt --project-ref dpcybjkruqrbdlyvfeqb`.
- `open-brain-enhanced` is registered in Claude Code (user scope, HTTP, `x-brain-key`). Direct JSON-RPC curl with the key from `~/.claude.json` works for verification when its tools aren't loaded.
- `gh`/`git`/`supabase`/`node` need the sandbox disabled on this machine (`apply-seccomp` nested-userns error).
- Deployed functions: open-brain-mcp (stock), enhanced-mcp, delete-thought-mcp, household-knowledge, home-maintenance, family-calendar, meal-planning. NOT deployed: rest-api, open-brain-rest, smart-ingest.

### Known gaps
- Re-scanned 2026-10-25: 49 of 814 `standard` rows matched the patterns, but nearly all were false positives (news imports; digit runs in URLs hitting `ssn_pattern`/`credit_card`). User chose to raise only 3 genuinely personal notes. `personal` is only a label today (nothing filters it). Never bulk-apply pattern matches.
- The stock connector re-classifies `type` on duplicate captures.
- `rest-api` / `smart-ingest` still have the UUID/capture problems (upstream #379, #504/#429/#367/#473). `open-brain-rest` uses a 0..100 importance scale (would be clamped to 6).
- `upsert_thought` and several other SECURITY INVOKER functions are still anon-executable (low risk while anon has no table privileges on `thoughts`).

### Re-check upstream (read-only)
```
gh pr view 490 --repo NateBJones-Projects/OB1 --json state,mergedAt
gh pr view 506 --repo NateBJones-Projects/OB1 --json state,mergedAt
gh api repos/NateBJones-Projects/OB1/commits/HEAD --jq '.sha[0:7] + " " + .commit.committer.date'
```
Last synced upstream HEAD: `238df6c` (2026-09-10). On the next sync, expect conflicts in `integrations/enhanced-mcp/index.ts`, `_shared/config.ts` copies, and `schemas/enhanced-thoughts/schema.sql` (`upsert_thought`); keep the local behavior unless upstream supersedes it.

## Security (corrected 2026-09-19)
- Aug 5 remediation (RLS + revokes on `crm-person-tiers`, `security_invoker=true` on `brain-health-monitoring` ops views) intact.
- enhanced-thoughts SECURITY DEFINER exposure (`brain_stats_aggregate`, `get_thought_connections`) fixed and verified live 2026-09-19 (`proacl` = owner + `service_role` only).
- **Rule:** revoke from `PUBLIC, anon, authenticated`; verify with `has_function_privilege()` and raw `proacl::text`, never a `pg_roles` join.
- **Rule:** before `CREATE OR REPLACE` on a live function, dump `pg_get_functiondef` and diff; repo files can lack live-only settings (e.g. `search_path`), and replacing drops them.
- Any `search_path` pin on a function using extension types must include `extensions`.
- Landmine catalog `.planning/supabase-security/landmine-catalog-2026-09-13.md` (with 2026-09-19 addendum). Read it before deploying any schema.

## Open items
1. Next capability from the queue (rest-api / smart-ingest blocked on UUID fixes).
2. Third upstream sync when upstream HEAD moves past `238df6c`.
3. `contrib/kae36/url-batch-import` still has 6 unmerged commits (unrelated URL-import recipe).

## Housekeeping
- Machine-local ignores live in `.git/info/exclude`; `.gitignore` matches upstream.
- The 721 thoughts with a null `type` column already carry types in stock metadata; `backfill_thought_types()` is moot unless something needs the typed column.
