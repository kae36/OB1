---
id: bring-my-open-brain-up-to-date
title: Bring My Open Brain Up To Date
status: active
created: 2026-08-01T06:28:46.561Z
updated: 2026-08-01T06:28:46.561Z
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
- Supabase project + all deployed Edge Functions are unaffected by which machine you work from — deployment is independent of any local `supabase/` folder (see `docs/01-getting-started.md`).
- `open-brain-credential-tracker.xlsx` holds original setup values (project URL, keys).
- **Claude Desktop is now installed on M7GASG5 and working**, after resolving a machine-hanging bug: the app's default OS-keyring integration (`libsecret`/gnome-keyring on XFCE) caused a full desktop freeze on first launch. Fix: launch with `claude-desktop --password-store=basic` (bypasses the OS keyring; tradeoff is the login session isn't persisted across app restarts — you'll need to re-sign-in each time it's relaunched this way).
- **User preference:** handles credential-sensitive setup (Supabase CLI login, secrets, deploy) themselves at the console — don't search for credential files or run CLI auth/deploy commands proactively; walk through steps conversationally instead.
- Fork's `main` mirror is fully synced with upstream (`6779106`) — no pending upstream commits to sync before adopting capabilities.

## Capability queue
Catalogued upstream additions available to adopt, in no particular order:
`delete-thought-mcp` (in progress, see below), `enhanced-mcp`, `hermes-agent-memory`, `rest-api`, `smart-ingest`, `chrome-capture-extension`, `consolidation-workers`, `brain-health-monitoring`, `gmail-smart-pull`, `brain-stats-daily`, `crm-person-tiers`, `auto-capture-claude-code` skill.

## Status: delete-thought-mcp
- Deployed to Supabase (confirmed live in dashboard).
- Registered as its own Claude Desktop custom connector (`?key=<MCP_ACCESS_KEY>` in the URL — not the OAuth Client ID/Secret fields, which don't apply to this key-based auth).
- **Not yet confirmed working conversationally.** Claude Desktop refused to call `delete_thought` when asked directly, citing an internal operating-rule against irreversible deletion — unclear if this is a soft "confirm first" heuristic (untested: a more explicit one-time-authorization phrasing) or a hard client-side policy.
- Manually verified the underlying delete works by deleting a test row directly via Supabase Table Editor.
- **Discovered a real gap in the core server** (`server/index.ts`): none of `capture_thought`, `list_thoughts`, `search_thoughts`, or `thought_stats` ever return a thought's `id` in their output — `capture_thought` even computes the id internally (line 473) but never returns it (line 487-497). This makes any id-requiring tool (like delete) unreachable through normal conversation without a manual DB lookup. Worth fixing regardless of the refusal question above.
- `integrations/delete-thought-mcp/*` files remain staged/uncommitted on `kae36/personal`, pending a decision on whether current state is "good enough" to commit as-is.

## Next steps (pick up here)
1. Decide on `delete-thought-mcp`: (a) retry the conversational call with explicit authorization wording to test if the refusal is soft, (b) patch `server/index.ts` to surface `id` in tool output and redeploy `open-brain-mcp` first, or (c) accept current state and commit as-is, treating direct Supabase access as the deletion fallback.
2. Once `delete-thought-mcp` is settled, move to the next capability in the queue — `brain-health-monitoring` was previously flagged as next in line.

See checkpoints `checkpoint_77914152`, `checkpoint_5225b9da`, `checkpoint_480401fe`, `checkpoint_d3d031da` for the full evidence trail.
