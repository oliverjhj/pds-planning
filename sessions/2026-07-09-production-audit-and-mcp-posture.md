# Session: 2026-07-09 — Production audit clean (item 4), MCP posture change, docs sweep

*(Second session of 2026-07-09; follows `2026-07-09-edge-function-sentry.md`.)*

**What changed:** Closed STATE item 4 in full, plus the MCP posture change that enabled it. Part (a) — the one-shot assume-nothing production audit: migration ledgers on production and staging are **identical** (22 files, 001–024 with the historic 012/013 gap, matching version timestamps), and migrations 020–024 were verified **by artefact** on BOTH environments (020 `operators_select_by_device` policy, 021 `handle_new_user` company-name gate, 022 `get_routes_since`, 023 `audio_version` emission, 024 `devices.android_id` unique index — all physically present). **Production matches the record; no remediation; the audit closed with zero findings.** Part (b) — the stale pair-device "deferred unique index" comment rewritten to state the current constraint. The enabling change: the single staging-pinned Supabase MCP was replaced (owner-executed, `claude mcp` remove/add) by **two permanently-pinned, permanently READ-ONLY servers** — `supabase-staging` / `supabase-prod` — so production is now readable (never writable) via MCP and the environment is explicit in every tool name. The owner floated MCP write-mode; counselled against (no speed benefit; would dissolve the human deploy gate) and the owner declined it — all writes remain human-gated. Both repo CLAUDE.mds' MCP sections brought current (clearing the banked stale item); dashboard CLAUDE.md deployment-state note and verification stamp refreshed. Finally, the **close-out methodology changed** (owner-directed): the workspace CLAUDE.md memory ritual now has a third step — Claude commits and pushes `pds-planning` automatically at session close (this commit is the first under that rule); the code-repo push gate is unchanged.

**Commits:** (all reviewed and pushed by the owner)
- pds-dashboard `b0538ba` — docs: fix stale pair-device comment — migration 024 landed the android_id unique index
- pds-dashboard `e75ac04` — docs: bring CLAUDE.md current — MCP posture (two read-only Supabase pins) + item-4 audit result
- pds-dashboard `15275ee` — docs: refresh the CLAUDE.md verification stamp to e75ac04
- pds-android `eedeb4f` — docs: bring the Available MCP Tools section current — seven servers, two read-only Supabase pins
- pds-planning (this commit) — session log + STATE.md (pushed by Claude per the new ritual step 3)

**Decisions made:**
- **Two permanent read-only Supabase MCP pins** (`supabase-staging` `febcpudiorwnsbcphcng` / `supabase-prod` `qopkjsihmdplmsoqrzji`) supersede "staging-pinned at rest + temporary re-point". Read-only is enforced at the MCP server layer on both. Ledger #22.
- **MCP write-mode declined** — owner offered; declined on counsel. All writes (migrations, deploys, MCP write-mode changes) stay human-gated. Unchanged posture, now explicitly re-affirmed.
- **Close-out auto-push for `pds-planning`** (owner-directed): the memory ritual's new step 3 — Claude stages, commits, and pushes the close-out. Deliberate exception to the human-push gate (documentation-only repo). Ledger #23.
- **Android CLAUDE.md verification stamp deliberately NOT refreshed** — the file was not re-verified against the driver-controls slice (`1585a47`); its Built Feature Conventions section doesn't cover that slice yet. Restamping would over-claim. Banked as one deliberate docs touch, best after item 5.
- Audit method precedent: verify migrations by **artefact** (pg_policies / pg_get_functiondef / pg_indexes probes), not ledger rows alone; verify MCP identity (`get_project_url`) before trusting any environment read.

**Verified:** MCP identity on both servers before any trusted read. Migration ledgers via `list_migrations` on both environments; artefact probes via read-only `execute_sql` on both. Repo migrations directory confirmed at the same 22 files. Dashboard `npm run typecheck` + `npm run build` clean before the one commit touching a `.ts` file (comment-only). No hardware involvement — nothing touched the compliance surface.

**What's next:** STATE item 5 — the consolidated GPS glass pass on the Lenovo tablet (route `575261eb` + Lockito): the four deferred diversion-skip cases, the manual-H&R fallback behaviours, and the tube-map H&R rendering eyeball. Last pre-pilot item before Stage 4 (Firebase split) and the pilot.

**Banked / open:**
- **Android CLAUDE.md single future docs touch:** fold the driver-controls/audio-output conventions into Built Feature Conventions, refresh the `835efd8` stamp, and fix its Setup Notes (~lines 179–180) which still call the item-4 audit "queued" (missed in this session's MCP-scoped sweep). Best after item 5.
- Deployed `pair-device` diverges from repo HEAD by the fixed comment only, until the next function deploy — accepted (recorded in `b0538ba`).
- ANDROID-3/4 Sentry issues still open (real July 1 tablet errors) — worth a look before the pilot.
- Pre-existing lint error `scripts/render-fixed-announcements.ts:172` — still banked.
- Render-worker latent gap (unchecked returned-`error` fields) — still banked.
- Remaining known-stale dashboard text (three sites): `src/proxy.ts` gating comment, routes empty-state, `triggerReRender` comment.
