---
id: bring-my-open-brain-up-to-date
title: Bring My Open Brain Up To Date
status: active
created: 2026-08-01T06:28:46.561Z
updated: 2026-08-02T00:15:37.153Z
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

## Environment (as of 2026-08-01)
- Working from the **M7GASG5 Ubuntu host console** now (moved off the Razer laptop, which originally hosted local Supabase CLI setup and is now low on disk).
- Supabase project + all deployed Edge Functions are unaffected by which machine you work from — deployment is independent of any local `supabase/` folder (see `docs/01-getting-started.md`). Note: `supabase/` is gitignored repo-wide — it's a local deploy staging dir only, never the source of truth (that's `server/` and `integrations/*/`).
- `open-brain-credential-tracker.xlsx` holds original setup values (project URL, keys).
- **Claude Desktop is now installed on M7GASG5 and working**, after resolving a machine-hanging bug: the app's default OS-keyring integration (`libsecret`/gnome-keyring on XFCE) caused a full desktop freeze on first launch. Fix: launch with `claude-desktop --password-store=basic` (bypasses the OS keyring; tradeoff is the login session isn't persisted across app restarts — you'll need to re-sign-in each time it's relaunched this way).
- **User preference:** handles credential-sensitive setup (Supabase CLI login, secrets, deploy) themselves at the console — don't search for credential files or run CLI auth/deploy commands proactively; walk through steps conversationally instead.
- Fork's `main` mirror is fully synced with upstream (`6779106`) — no pending upstream commits to sync before adopting capabilities.
- No local Docker/Podman on M7GASG5 — `supabase db dump`/`db diff` and CLI functions deploy's docker path don't work. Workaround that does work: `supabase functions download --use-api` for function source, and the Management API (`POST /v1/projects/{ref}/database/query` with the CLI's stored `~/.supabase/access-token`) for arbitrary SQL (read or write) against the live DB — this is now the standard way to inspect/migrate the DB from this environment.

## Capability queue
Catalogued upstream additions available to adopt, in no particular order:
`delete-thought-mcp` (**done**), `brain-health-monitoring` (**done**, see below), `enhanced-mcp`, `hermes-agent-memory`, `rest-api`, `smart-ingest`, `chrome-capture-extension`, `consolidation-workers`, `gmail-smart-pull`, `brain-stats-daily`, `crm-person-tiers`, `auto-capture-claude-code` skill.

## Status: delete-thought-mcp — DONE (2026-08-01)
- Deployed to Supabase, registered as a Claude Desktop custom connector (`?key=<MCP_ACCESS_KEY>` in the URL — not OAuth fields), verified working by direct Supabase-side delete of a test row.
- `integrations/delete-thought-mcp/*` committed (`bf6f203`) and pushed to `origin/kae36/personal`. Git, deployed function, and DB are all consistent for this integration.
- **Still open, deferred, not blocking:** (a) Claude Desktop refused to call `delete_thought` conversationally, citing an internal operating-rule against irreversible deletion — untested whether more explicit one-time-authorization phrasing changes that. (b) `server/index.ts`'s `capture_thought`/`list_thoughts`/`search_thoughts`/`thought_stats` still never return a thought's `id` in output (`capture_thought` computes it at line 473 but never returns it) — makes any id-requiring tool unreachable through normal conversation without a manual DB lookup. Pick up here if you want `delete_thought` to be usable hands-free.

## Status: prod/git drift audit — DONE (2026-08-01)
- Audited all 6 deployed Edge Functions against local `kae36/personal` HEAD, plus the live DB schema against repo docs.
- Found `open-brain-mcp` stale in prod (running a 2026-03-30 commit, 2 commits behind HEAD — missing ChatGPT `search`/`fetch` compat tools, JSON-RPC auth-error envelope, tool `annotations`). Redeployed from HEAD (`a42695f`), confirmed byte-identical post-deploy.
- All 6 deployed functions + DB schema + git (`bf6f203`, pushed) confirmed in sync as of that audit.

## Status: brain-health-monitoring — DONE (2026-08-02)
- Pulled `recipes/brain-health-monitoring` from `origin/main`, deployed to live Supabase.
- **Found an undocumented cross-schema dependency bug while doing this**: the recipe's stated prerequisite, `schemas/enhanced-thoughts`, replaces `upsert_thought()` with a version that writes to `status`/`status_updated_at` columns — but never creates them itself. Those only exist via `schemas/workflow-status`, which isn't mentioned anywhere in `enhanced-thoughts`'s Prerequisites. Installing `enhanced-thoughts` alone would have broken `capture_thought` on the very next call.
- Installed in the correct order: `schemas/enhanced-thoughts` → `schemas/workflow-status` → `recipes/brain-health-monitoring/ops-views.sql`. Smoke-tested `upsert_thought()` post-install (insert + delete of a throwaway row) to confirm no runtime error.
- 5 of 8 `ops_*` views created (`ops_source_volume_24h`, `ops_recent_thoughts`, `ops_enrichment_gaps`, `ops_type_distribution`, `ops_sensitivity_distribution`); the other 3 need `smart-ingest`/`entity-extraction` (not adopted, skipped cleanly by the SQL's own guards — by design).
- **Known data quirk, not a bug:** `ops_type_distribution` shows 721/~770 thoughts as `unclassified`. The `enhanced-thoughts` backfill only promotes `metadata->>'type'` into the new `type` column for a specific enum list (`idea`, `task`, `person_note`, `reference`, `decision`, `lesson`, `meeting`, `journal`); most historical thoughts used other type strings (e.g. `observation`), so their data lives in `metadata` but isn't column-promoted. Don't read `ops_type_distribution` as "most thoughts have no type" — they do, just not in the promoted column.
- Committed + pushed to `origin/kae36/personal`.

## Next steps (pick up here)
1. Pick the next capability from the queue — no strong prior lean; candidates worth a fresh look are `brain-stats-daily` (small, likely pairs well with the ops views just installed) or `enhanced-mcp` (bigger lift, upgrades MCP surface 4→13 tools).
2. Optional, low-priority: revisit the two deferred `delete_thought` items (id-surfacing gap in `server/index.ts`, conversational-refusal question) if hands-free deletion becomes worth the effort.
3. Optional: the `ops_type_distribution` unclassified-majority quirk could be fixed with a one-off backfill UPDATE that copies any non-null `metadata->>'type'` into the `type` column regardless of enum membership — not done, just noting it's easy if wanted later.

See checkpoints `checkpoint_77914152`, `checkpoint_5225b9da`, `checkpoint_480401fe`, `checkpoint_d3d031da`, `checkpoint_e1bf4eff`, `checkpoint_40a90103` for the full evidence trail.
