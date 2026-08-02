---
id: bring-my-open-brain-up-to-date
title: Bring My Open Brain Up To Date
status: active
created: 2026-08-01T06:28:46.561Z
updated: 2026-08-02T00:24:43.394Z
tags:
  - open-brain
  - upstream-adoption
  - supabase
  - m7gasg5
  - mcp-connectors
---

# Bring My Open Brain Up To Date

## Goal
Adopt useful new upstream OB1 capabilities into my personal Open Brain instance (Supabase project `dpcybjkruqrbdlyvfeqb`, "kae36's Org"), one capability at a time — deploy, register as a Claude Desktop connector, verify it actually works, then move to the next.

## Environment (as of 2026-08-02)
- Working from the **M7GASG5 Ubuntu host console** now (moved off the Razer laptop, which originally hosted local Supabase CLI setup and is now low on disk).
- Supabase project + all deployed Edge Functions are unaffected by which machine you work from — deployment is independent of any local `supabase/` folder (see `docs/01-getting-started.md`). Note: `supabase/` is gitignored repo-wide — it's a local deploy staging dir only, never the source of truth (that's `server/` and `integrations/*/`).
- `open-brain-credential-tracker.xlsx` holds original setup values (project URL, keys).
- **Claude Desktop is now installed on M7GASG5 and working**, after resolving a machine-hanging bug: launch with `claude-desktop --password-store=basic` (bypasses the OS keyring; tradeoff is the login session isn't persisted across app restarts).
- **User preference:** handles credential-sensitive setup (Supabase CLI login, secrets, deploy) themselves at the console — don't search for credential files or run CLI auth/deploy commands proactively; walk through steps conversationally instead.
- Fork's `main` mirror is fully synced with upstream (`6779106`) — no pending upstream commits to sync before adopting capabilities.
- No local Docker/Podman on M7GASG5 — `supabase db dump`/`db diff` and CLI functions deploy's docker path don't work. Workaround: `supabase functions download --use-api` for function source, and the Management API (`POST /v1/projects/{ref}/database/query` with the CLI's stored `~/.supabase/access-token`) for arbitrary SQL (read or write) against the live DB — this is now the standard way to inspect/migrate the DB from this environment.
- **Schema state on the live DB now includes:** core thoughts table + `content_fingerprint` dedup, `schemas/enhanced-thoughts` (adds `type`/`sensitivity_tier`/`importance`/`quality_score`/`source_type`/`enriched` columns + 3 RPCs), `schemas/workflow-status` (adds `status`/`status_updated_at`), `recipes/brain-health-monitoring` (5 `ops_*` views), `schemas/brain-stats-daily` (4 heatmap RPCs), `schemas/crm-person-tiers` (`crm_persons` + `crm_person_mentions` tables, `crm_person_tiers()` RPC — empty, no persons populated yet). Check this list before assuming a schema/column is or isn't installed — keep it updated as more get adopted.
- **Deploy playbook that's worked 4x now:** (1) `git checkout origin/main -- <path>` to pull the capability into the working tree, (2) read README + schema.sql fully, checking stated Prerequisites *and* grepping the SQL body for column/table references the file itself doesn't create (enhanced-thoughts's undocumented workflow-status dependency is why), (3) apply via the Management API, (4) verify objects exist + smoke-test with a real insert/query/cleanup cycle, (5) checkpoint, update this brief, commit, ask before pushing.

## Capability queue
`delete-thought-mcp` (**done**), `brain-health-monitoring` (**done**), `brain-stats-daily` (**done**), `crm-person-tiers` (**done**, see below), `enhanced-mcp`, `hermes-agent-memory`, `rest-api`, `smart-ingest`, `chrome-capture-extension`, `consolidation-workers`, `gmail-smart-pull`, `auto-capture-claude-code` skill.

## Status: delete-thought-mcp — DONE (2026-08-01)
- Deployed, connector registered, committed (`bf6f203`) + pushed.
- **Still open, deferred, not blocking:** (a) Claude Desktop refused to call `delete_thought` conversationally citing an operating-rule — untested whether explicit authorization wording changes that. (b) `server/index.ts` still never returns a thought's `id` from `capture_thought`/`list_thoughts`/`search_thoughts`/`thought_stats` — makes id-requiring tools unreachable conversationally without a manual DB lookup.

## Status: prod/git drift audit — DONE (2026-08-01)
- Found + fixed `open-brain-mcp` running 2 commits stale in prod. Redeployed from HEAD, confirmed byte-identical. All 6 functions + DB schema + git now stay in sync as new capabilities are adopted (re-verify after each future deploy).

## Status: brain-health-monitoring — DONE (2026-08-02)
- Deployed `enhanced-thoughts` → `workflow-status` → `ops-views.sql`, in that order, after discovering `enhanced-thoughts`'s undocumented dependency on `workflow-status`'s `status` columns (see playbook step 2 above — this is *why* that step exists now).
- 5 of 8 `ops_*` views created; 3 need `smart-ingest`/`entity-extraction` (not adopted).
- **Known data quirk, not a bug:** most historical thoughts show `unclassified` in `ops_type_distribution` — backfill only promotes `metadata->>'type'` for a specific enum list, older data still lives in `metadata`.
- Committed (`f0f82df`) + pushed.

## Status: brain-stats-daily — DONE (2026-08-02)
- 4 heatmap RPCs, no dependency surprises. Smoke-tested 30-day and 365-day windows against real data. Committed (`859d60f`) + pushed.

## Status: crm-person-tiers — DONE (2026-08-02)
- New standalone `crm_persons` + `crm_person_mentions` tables (core `thoughts` untouched), `crm_person_tiers()` RPC with 4-tier taxonomy (`connected`/`contact`/`known`/`unknown`) and activity-based auto-promotion.
- No hidden dependencies. Full round-trip smoke test: inserted a test person, confirmed tier/mention_count via the RPC, deleted the test row.
- Table is empty — no real persons populated yet; that's a separate future task (manual entry or an import script) if you want to actually use it.
- Not yet committed/pushed — do that next if continuing straight through.

## Next steps (pick up here)
1. Commit + push `schemas/crm-person-tiers/*` (currently staged locally, uncommitted).
2. Pick the next capability. `enhanced-mcp` is the biggest remaining lift (4→13 tools) — worth reading its README closely before committing to it, given the pattern of undocumented dependencies found so far. `gmail-smart-pull` or `chrome-capture-extension` might be smaller, more self-contained next steps if you want to keep the streak of quick wins going.
3. Optional, low-priority: revisit the two deferred `delete_thought` items (id-surfacing gap, conversational-refusal question).
4. Optional: populate `crm_persons` with real data if the CRM layer is actually going to get used, and/or wire up its dashboard snippet.
5. Optional: fix the `ops_type_distribution` unclassified-majority quirk with a one-off backfill UPDATE.

See checkpoints `checkpoint_77914152`, `checkpoint_5225b9da`, `checkpoint_480401fe`, `checkpoint_d3d031da`, `checkpoint_e1bf4eff`, `checkpoint_40a90103`, `checkpoint_9b2f50ae`, `checkpoint_dcf384ef` for the full evidence trail.
