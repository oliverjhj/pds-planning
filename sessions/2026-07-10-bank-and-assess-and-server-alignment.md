# Session: 2026-07-10 — Bank-and-Assess sweep, server-alignment triage, active_route_id slice

**What changed:** Three work efforts in one session, clearing nearly the whole pre-pilot odds-and-ends backlog.

1. **Bank-and-Assess sweep (8 commits).** The render worker now checks the returned `error` of all three queue/status writes (supabase-js returns PostgREST errors rather than throwing, so the old try/catches were dead for that class) — redeployed to staging AND production same day, verified byte-identical by digest. The two real Sentry crashes fixed: ANDROID-4 (`REQUEST_INSTALL_PACKAGES` declared + guarded `canRequestPackageInstalls()`; glass-verified on the tablet) and ANDROID-3 (bounded 3-attempt retry around the Tink keyset build; the permanent-loss vector was already dead via the B1 `allowBackup=false` fix). The Rule-17 Android PII scrubber is now real (mirrors the edge `scrub.ts` semantics onto the sentry-java 8.7.0 typed event model; hex threshold 40 regression-guarded; 16 JUnit tests) plus a new `beforeBreadcrumb` hook. `journey_skipped_stops` is cleared on launch-recovery auto-clear. Dashboard lint is fully green, the three known-stale text sites are fixed (CLAUDE.md gotcha retired), and the Android CLAUDE.md logcat filter names the real dotted tags. Both ANDROID-3/4 marked resolved in Sentry with fixing commits in the activity feed.
2. **Server-alignment triage.** The "~9 hardening items" list didn't survive the workflow migration, so it was re-derived by a two-agent diff of the frozen Data-Architecture spec against the deployed implementation. Verdict: every hardening divergence re-banked as not-pilot-visible (full table now in STATE.md Defer section); one adjacent item promoted and **built** — `devices.active_route_id`.
3. **active_route_id slice (glass-verified).** SyncManager stage 7 (the reserved no-op) now PATCHes `devices.active_route_id` from `journey_state` statelessly every sync via a new `DeviceRowRepository.patchActiveRoute` (explicit `JsonNull` for journey-end); a new `TriggerJourneySyncUseCase` (the `ForceSyncUseCase` shape — ViewModels stay use-cases-only) fires `journey_start`/`journey_end` through the gated `SyncTriggers` entry at all four journey seams. The dashboard fleet view's previously-always-empty "active route" column now works. Bonus proven live: journey summaries upload sub-second after journey end (previously up to 30 min on the periodic).
4. **Ops items.** Sentry spike protection confirmed enabled on all three projects; a new `edge-functions error flood` metric monitor (count > 50/hr → High-priority issue → the project's email rule, connected explicitly; email delivery empirically proven from the Jul 9 inbox). Supabase CLI devDependency bumped 2.100.0 → 2.109.1.

**Commits:**
- pds-dashboard `0abdf07` — fix: audio-render-worker checks returned errors from queue RPCs and the failed-status stamp (deployed both envs 2026-07-10)
- pds-dashboard `bff651b` — chore: fix prefer-const lint error in render-fixed-announcements script
- pds-dashboard `68df64c` — docs: stale-text sweep (three sites) + CLAUDE.md gotcha retirement
- pds-dashboard `a39cf98` — chore: bump supabase CLI devDependency 2.100.0 → 2.109.1
- pds-android `b831b00` — fix: declare REQUEST_INSTALL_PACKAGES and guard the maintenance-exit installer check (ANDROID-4)
- pds-android `6ba13cf` — fix: bounded retry around Tink keyset provisioning against transient Keystore failures (ANDROID-3)
- pds-android `28d745a` — fix: clear journey_skipped_stops on launch-recovery auto-clear (FR-AT-18)
- pds-android `259a168` — feat: implement the Rule 17 / NFR-R-07 Sentry PII scrubber
- pds-android `1d9ed52` — docs: fix the CLAUDE.md logcat filter to the real dotted tags
- pds-android `684879d` — feat: report devices.active_route_id via sync stage 7 + journey sync triggers

(All reviewed and pushed by owner; `audio-render-worker` redeployed by owner to staging + production.)

**Decisions made:**
- **Server-alignment triage verdicts (the "triage, don't clear" instruction executed):** all hardening divergences re-banked with reasons — notably recover-device stays deliberately MORE permissive than spec (the spec'd 15-min lockout could brick a flaky pilot tablet), and the `devices_update_by_device` WITH CHECK tightening stays a deliberate post-pilot production-migration task. FCM payload naming closed as an accepted divergence (both ends agree; docs are evidence). Full table in STATE.md.
- **active_route_id design:** journey lifecycle stays Room-only (Rules 2/3) — network write lives in sync stage 7, stateless every sync; freshness from journey-seam sync triggers; a `journey_end` trigger dropped as `AlreadyRunning` is accepted (self-corrects on any later sync; drop-not-queue is load-bearing). Trigger wiring via a thin domain wrapper (`TriggerJourneySyncUseCase`) following the ForceSyncUseCase precedent rather than direct `SyncTriggers` injection into ViewModels.
- **ANDROID-3 scope:** transient-class retry only; the deep self-heal (permanent key loss → recreate keyset → pairing screen instead of crash-loop) deliberately not built — banked post-pilot.
- **D1 control flow unchanged:** a failed job-completion still returns 200 after a successful render (files are in Storage; pgboss-maintain resets a stuck job; differential re-render makes the retry cheap) — observe, don't abort.

**Verified:** Dashboard — vitest 14/14, lint fully green, typecheck, production build; worker deploys verified byte-identical (same ezbr_sha256) on both projects and a live render smoke passed post-deploy. Android — full unit suite + `assembleStagingDebug` green after every commit. **Glass (owner, tablet + staging):** ANDROID-4 maintenance exit no longer crashes (Settings + unknown-sources screen open as designed); active_route_id set on journey start and NULL after end (confirmed server-side via MCP at both states); journey summaries uploaded sub-second after three consecutive journey ends (14:12/14:13/14:14 UTC — versus 22 min pre-change). Return-journey flip not glass-tested (test route has no return route configured — same composed code path; left to iterative testing).

**What's next:** The pilot — owner-led iterative testing. No numbered build items and no banked pre-pilot work remain.

**Banked / open:**
- Repo CLAUDE.md staleness nits from today's work (sweep opportunistically, docs commits): Android CLAUDE.md still calls sync stage 7 "the one remaining reserved no-op slot"; dashboard CLAUDE.md deployment stamp says all functions deployed 2026-07-09 (audio-render-worker is now 2026-07-10 HEAD).
- Fleet-view "active route" cell is request-time rendered — manual refresh needed to see tablet-side changes. Auto-refresh (polling or Supabase realtime) is a small future dashboard feature, only if the pilot wants it.
- `journey_end` trigger can drop as `AlreadyRunning` → stale fleet cell ≤30 min — accepted by design; revisit only if operationally annoying.
- Sentry hygiene (optional): the three default alert rules are identically named ("Send a notification for high priority issues") — rename to include project names to avoid future ambiguity.
- Stale banked items removed from STATE.md this session: the stationary-detection timeout was already operator-configurable (`AutoTimeoutPolicy`/FR-AT-49 — the banked note was stale), and the 2026-07-10 journey-summary cross-check came back clean (6/6 announced, zero anomalies).
