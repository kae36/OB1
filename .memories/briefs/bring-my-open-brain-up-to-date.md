---
id: bring-my-open-brain-up-to-date
title: Bring My Open Brain Up To Date
status: active
created: 2026-08-01T06:28:46.561Z
updated: 2026-08-02T00:21:30.365Z
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
- **Claude Desktop is now installed on M7GASG5 and working**, after resolving a machine-hanging bug: the app's default OS-keyring integration (`libsecret`/gnome-keyring on XFCE) caused a full desktop freeze on first launch. Fix: launch with `claude-desktop --password-store=basic` (bypasses the OS keyring; tradeoff is the login session isn't persisted across app restarts — you'll need to re-sign-in each time it's relaunched this way).
- **User preference:** handles credential-sensitive setup (Supabase CLI login, secrets, deploy) themselves at the console — don't search for credential files or run CLI auth/deploy commands proactively; walk through steps conversationally instead.
- Fork's `main` mirror is fully synced with upstream (`6779106`) — no pending upstream commits to sync before adopting capabilities.
- No local Docker/Podman on M7GASG5 — `supabase db dump`/`db diff` and CLI functions deploy's docker path don't work. Workaround that does work: `supabase functions download --use-api` for function source, and the Management API (`POST /v1/projects/{ref}/database/query` with the CLI's stored `~/.supabase/access-token`) for arbitrary SQL (read or write) against the live DB — this is now the standard way to inspect/migrate the DB from this environment.
- **Schema state on the live DB now includes:** core thoughts table + `content_fingerprint` dedup, `schemas/enhanced-thoughts` (adds `type`/`sensitivity_tier`/`importance`/`quality_score`/`source_type`/`enriched` columns + 3 RPCs), `schemas/workflow-status` (adds `status`/`status_updated_at`), `recipes/brain-health-monitoring` (5 `ops_*` views), `schemas/brain-stats-daily` (4 heatmap RPCs). Check this list before assuming a schema/column is or isn't installed — keep it updated as more get adopted.

## Capability queue
Catalogued upstream additions available to adopt, in no particular order:
`delete-thought-mcp` (**done**), `brain-health-monitoring` (**done**), `brain-stats-daily` (**done**, see below), `enhanced-mcp`, `hermes-agent-memory`, `rest-api`, `smart-ingest`, `chrome-capture-extension`, `consolidation-workers`, `gmail-smart-pull`, `crm-person-tiers`, `auto-capture-claude-code` skill.

## Status: delete-thought-mcp — DONE (2026-08-01)
- Deployed to Supabase, registered as a Claude Desktop custom connector (`?key=<MCP_ACCESS_KEY>` in the URL — not OAuth fields), verified working by direct Supabase-side delete of a test row.
- `integrations/delete-thought-mcp/*` committed (`bf6f203`) and pushed to `origin/kae36/personal`.
- **Still open, deferred, not blocking:** (a) Claude Desktop refused to call `delete_thought` conversationally, citing an internal operating-rule against irreversible deletion — untested whether more explicit one-time-authorization phrasing changes that. (b) `server/index.ts`'s `capture_thought`/`list_thoughts`/`search_thoughts`/`thought_stats` still never return a thought's `id` in output — makes any id-requiring tool unreachable through normal conversation without a manual DB lookup.

## Status: prod/git drift audit — DONE (2026-08-01)
- Audited all 6 deployed Edge Functions + live DB schema against local `kae36/personal` HEAD / repo docs.
- Found and fixed `open-brain-mcp` running 2 commits stale in prod (missing ChatGPT compat tools, JSON-RPC auth envelope, tool annotations). Redeployed from HEAD, confirmed byte-identical.
- All 6 functions + DB schema + git now stay in sync as new capabilities are adopted (re-verify after each future deploy).

## Status: brain-health-monitoring — DONE (2026-08-02)
- Deployed `schemas/enhanced-thoughts` → `schemas/workflow-status` → `recipes/brain-health-monitoring/ops-views.sql`, in that order.
- **Found an undocumented cross-schema dependency bug**: `enhanced-thoughts`'s rewritten `upsert_thought()` needs `workflow-status`'s `status`/`status_updated_at` columns, which its own Prerequisites section never mentions. Installing it alone would have broken `capture_thought` on the next call. Worth keeping in mind for any *other* schema whose Prerequisites section might be similarly incomplete — don't fully trust a stated prereq list; grep the SQL body for column references not created by that same file.
- 5 of 8 `ops_*` views created; the other 3 need `smart-ingest`/`entity-extraction` (not adopted, skipped cleanly by design).
- **Known data quirk, not a bug:** `ops_type_distribution` shows most historical thoughts as `unclassified` — the backfill only promotes `metadata->>'type'` into the column for a specific enum list; older thoughts using other type strings (e.g. `observation`) keep their data in `metadata`, just not column-promoted.
- Committed (`f0f82df`) + pushed.

## Status: brain-stats-daily — DONE (2026-08-02)
- Deployed `schemas/brain-stats-daily/schema.sql` directly — no dependency surprises this time (its only prereq, `enhanced-thoughts`, was already installed).
- Adds 4 RPCs + 1 helper: `brain_stats_daily`, `brain_stats_daily_lifelog`, `brain_stats_daily_jsonb`, `brain_stats_daily_lifelog_jsonb`. All `SECURITY INVOKER`, granted to `authenticated`/`service_role` only (not `anon` — safer default, deliberate per the schema's own docs).
- Smoke-tested: 30-day window returned real recent capture data; 365-day JSONB variant returned 23 days with activity.
- Optional dashboard wiring (`HeatmapSourceFilter.tsx`) not done — only relevant if `open-brain-dashboard-next` gets adopted later.
- Not yet committed/pushed — do that next if continuing straight through.

## Next steps (pick up here)
1. Commit + push `schemas/brain-stats-daily/*` (currently staged locally, uncommitted).
2. Pick the next capability from the queue. `crm-person-tiers` (schema, likely small) may be a natural next pairing with the stats/health work; `enhanced-mcp` is the biggest lift (4→13 tools) if there's appetite for it.
3. Optional, low-priority: revisit the two deferred `delete_thought` items (id-surfacing gap, conversational-refusal question).
4. Optional: the `ops_type_distribution` unclassified-majority quirk could be fixed with a one-off backfill UPDATE — not done, easy if wanted later.

See checkpoints `checkpoint_77914152`, `checkpoint_5225b9da`, `checkpoint_480401fe`, `checkpoint_d3d031da`, `checkpoint_e1bf4eff`, `checkpoint_40a90103`, `checkpoint_9b2f50ae` for the full evidence trail.
