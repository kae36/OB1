---
id: bring-my-open-brain-up-to-date
title: Bring My Open Brain Up To Date
status: active
created: 2026-08-01T06:28:46.561Z
updated: 2026-10-25T19:35:48.638Z
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

## Status as of 2026-10-25: enhanced-mcp LIVE with local patches

**Decision 2026-10-25 (user): stop waiting on upstream; apply PR #490 + #506 locally.** Both PRs still open/unmerged upstream (#490 untouched since 2026-09-02).
- Applied to `integrations/enhanced-mcp/index.ts`: all of #490; #506's import cleanup + `update_thought` removal. #506's capture/type hunks skipped (conflict; #490's version is a superset that also persists the embedding). README updated.
- Deploy copy lives at `supabase/functions/enhanced-mcp/` (gitignored, a COPY, not a symlink): after editing `integrations/enhanced-mcp/index.ts`, `cp` it there before `supabase functions deploy enhanced-mcp --no-verify-jwt --project-ref dpcybjkruqrbdlyvfeqb`.
- `open-brain-enhanced` re-registered (user scope, HTTP, `x-brain-key`).
- **Verified live 2026-10-25** (direct JSON-RPC curl to the function): tools/list shows no `update_thought`, `get_thought` id is uuid; semantic + text search + `search_thoughts_text` return hits; `get_thought` by UUID works; capture returns success and the row is found by semantic search (0.815) so the embedding is stored. Test thought deleted.
- When upstream merges #490/#506, the next upstream sync should conflict-free or trivially resolve to their version.

### Remaining known gaps (DB-side, not fixed by the PRs)
Live `upsert_thought` returns `{id, fingerprint}` and ignores top-level `p_payload` fields, so detected sensitivity tier / importance / quality are NOT persisted on enhanced captures (rows stay `standard`/50/70). Restricted content is still blocked at capture. Fix would need a DB function change (read those fields from top-level payload) — not done. `related_thoughts` integer-id concern from #506 not re-checked. `rest-api` and `smart-ingest` still have the UUID/capture problems (upstream #379, #504/#429/#367/#473); not adopted.

### Re-check upstream (read-only)
```
gh pr view 490 --repo NateBJones-Projects/OB1 --json state,mergedAt
gh pr view 506 --repo NateBJones-Projects/OB1 --json state,mergedAt
gh api repos/NateBJones-Projects/OB1/commits/HEAD --jq '.sha[0:7] + " " + .commit.committer.date'
```
Last synced upstream HEAD: `238df6c` (2026-09-10). `gh`/`git`/most shell commands need the sandbox disabled on this machine (`apply-seccomp` nested-userns error).

## Security (corrected 2026-09-19)
- Aug 5 remediation (RLS + revokes on `crm-person-tiers`, `security_invoker=true` on `brain-health-monitoring` ops views) intact.
- enhanced-thoughts SECURITY DEFINER exposure (`brain_stats_aggregate`, `get_thought_connections`) fixed and verified live 2026-09-19 (`proacl` = owner + `service_role` only).
- **Rule:** revoke from `PUBLIC, anon, authenticated`; verify with `has_function_privilege()` and raw `proacl::text`, never a `pg_roles` join. Repo schema REVOKEs naming only `anon` have the same gap.
- Any `search_path` pin on a function using extension types must include `extensions`.
- Landmine catalog `.planning/supabase-security/landmine-catalog-2026-09-13.md` (with 2026-09-19 addendum). Read it before deploying any schema.
- Lower priority: several SECURITY INVOKER functions still anon-executable; low risk while anon has no table privileges on `thoughts`.

## Housekeeping
- Machine-local ignores live in `.git/info/exclude`; `.gitignore` matches upstream.
- The 721 thoughts with a null `type` column already carry types in stock metadata; `backfill_thought_types()` is moot unless something needs the typed column.

## Open items
1. Optional: DB patch so `upsert_thought` persists sensitivity tier / importance / quality from enhanced captures.
2. Next capability from the queue (rest-api / smart-ingest blocked on UUID fixes).
3. Third upstream sync when upstream HEAD moves past `238df6c`.
4. `contrib/kae36/url-batch-import` still has 6 unmerged commits (unrelated URL-import recipe).

## Key files
- `.planning/supabase-security/` (gitignored, local): fix scripts and landmine catalog.
- User does deploys and secrets at the console. No local psql; dashboard SQL editor shows only the last statement's result.
