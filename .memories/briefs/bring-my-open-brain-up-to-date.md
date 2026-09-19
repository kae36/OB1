---
id: bring-my-open-brain-up-to-date
title: Bring My Open Brain Up To Date
status: active
created: 2026-08-01T06:28:46.561Z
updated: 2026-09-13T07:24:57.463Z
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
Adopt useful new upstream OB1 capabilities into my personal Open Brain instance (Supabase project `dpcybjkruqrbdlyvfeqb`, "kae36's Org") -- but as of 2026-09-13, the strategy changed: instead of cherry-picking capabilities one at a time, fully sync this fork with upstream first, reconcile security across everything that syncs in, and only then pick new capabilities to adopt.

## Status as of 2026-09-13 end of session

**Security: fully caught up and verified live.**
- The original Aug 5 remediation (RLS + revokes on `crm-person-tiers`, `security_invoker=true` on `brain-health-monitoring` ops views) is intact and was never at risk.
- Found and fixed a NEW live exposure in `schemas/enhanced-thoughts/schema.sql`: `brain_stats_aggregate` and `get_thought_connections` were `SECURITY DEFINER` + granted to `anon`, letting anyone with the public anon key bypass RLS and read restricted-tier thought content. Fixed and verified live (`anon`/`authenticated` revoked, only `service_role` + owner remain).
- Also pinned `search_path` on 6 functions flagged by the advisor for `function_search_path_mutable`.
- **Self-inflicted regression, found and fixed same session:** pinning `match_thoughts`'s search_path to `public, pg_temp` broke semantic search, because pgvector's `<=>` operator and `vector` type live in this project's `extensions` schema, not `public`. Fixed by adding `extensions` back to that one function's search_path. Verified live: semantic search works again. **Lesson recorded:** any future search_path pinning on a function touching extension-provided types (vector, pg_trgm, etc.) must include `extensions`, not just `public`.

**Upstream sync: done.** Merged `origin/main` (~350 commits, all of upstream `NateBJones-Projects/OB1`'s history since this fork diverged) into `kae36/personal`, preserving both security fixes. Pushed to GitHub. Every capability in the old adoption queue (`enhanced-mcp`, `hermes-agent-memory`, `rest-api`, `smart-ingest`, `chrome-capture-extension`, `consolidation-workers`, `gmail-smart-pull`, `auto-capture-claude-code`) now physically exists in the repo tree, current with upstream -- none are deployed to the live DB yet.

**New reference artifact:** `.planning/supabase-security/landmine-catalog-2026-09-13.md` -- audits all 12 `SECURITY DEFINER` functions and every `anon` grant across the ENTIRE merged codebase (not just deployed schemas). One more not-yet-deployed landmine found: `schemas/readwise-books/schema.sql` has 2 anon-granted RPCs (one a write path) and no RLS/REVOKE on its own table at all. Read this catalog before deploying any new schema.

**Todo-list workflow confirmed working:** thoughts schema already supports task tracking natively -- capture with natural phrasing ("Work task: ..." / "Home task: ..."), auto-tagged `type: task` with a `topics` array doing the work/home split (no new schema needed). Verified end-to-end: captured one work task (find a PTO day) and one home task (install browser39 on m7gasg5), both retrievable via semantic search and correctly separated by topic.

## Open items for next session
1. **`.gitignore`/ScratchPad discussion** -- user wanted to discuss how to handle the `/ScratchPad` gitignore entry after the merge; this got deferred and never circled back to. Ask again.
2. Decide the next capability to adopt from the now-current queue -- check the landmine catalog first for whichever one is picked, since 2 of ~14 audited functions across the whole codebase are still landmines (`editorial-policy`'s `get_recent_audit_reports`, and `readwise-books`'s two anon grants) -- neither is deployed, so no urgency unless one of those specific schemas is the next pick.
3. Housekeeping: `contrib/kae36/url-batch-import` branch still has 6 unmerged commits (a separate, unrelated URL-import recipe) -- untouched all session, still outstanding whenever it's wanted.
4. Pre-existing uncommitted `.memories` items from before this session started were absorbed into the merge; no longer an open item.

## Key files
- `.planning/supabase-security/` -- all fix scripts and the landmine catalog (gitignored, local only, not part of the public contribution contract)
- Deploy playbook and machine/environment notes from the original brief still apply unchanged (M7GASG5 console, Claude Desktop launch flag, dashboard SQL editor quirks, no local psql, etc.)

See checkpoints from 2026-09-13 for the full evidence trail of today's session (security fix, upstream sync plan/execution, search_path regression+fix, todo-list demo).
