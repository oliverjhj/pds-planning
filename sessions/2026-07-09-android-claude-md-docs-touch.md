# Session: 2026-07-09 — Android CLAUDE.md docs touch (banked item pulled forward)

*(Third session of 2026-07-09; follows `2026-07-09-production-audit-and-mcp-posture.md`.)*

**What changed:** The banked Android CLAUDE.md docs touch, pulled forward ahead of item 5 at the owner's direction. An Explore pass first verified every driver-controls/audio-output ledger decision (#9–15, #17–19) against the actual code — **all honoured** (gate ordering in `AnnouncementPlayer`, the state machines, the suppression call sites, the focus/keep-alive paths). Then one docs commit to `pds-android/CLAUDE.md`: a fifth Built Feature Conventions subsection ("Driver controls & audio output", FR-AT-29/30/31/44 + manual H&R FR-AT-26/41); the verification stamp refreshed `835efd8` → `eedeb4f` — honest now that the file actually covers the slice (the over-claim concern that had deferred it); the Setup Notes text still calling the item-4 audit "queued" replaced with the clean 2026-07-09 audit result; and the Project Structure tree gained the slice's classes (`domain/audio/`, `ManualHailRideState`, `DriverVolumeController`, `ExternalAudioSinks`). Two code-vs-ledger wording precisions surfaced and were stated as code reality (see Decisions).

**Commits:**
- pds-android `cdb4f3e` — docs: bring CLAUDE.md current with the driver-controls/audio-output slice (**pushed by Claude at the owner's explicit direction — a one-off exception to the human-push gate; docs-only diff shown in full in-session, plan approved in plan mode**)
- pds-planning (this commit) — session log + STATE.md close-out (pushed by Claude per ritual step 3)

**Decisions made:**
- **Coverage and restamp together resolves the over-claim concern.** Restamping the CLAUDE.md is honest once the file covers the slice and its claims are code-verified; glass-verification status stays single-homed in STATE.md item 5 — no transient "glass pending" note in CLAUDE.md (it would rot the moment item 5 completes).
- **No new numbered rules** — conventions only, per the frozen-rule-numbers discipline (ledger #16); the compliance-bearing points cite Rules 10/19 inline, matching how the slice code itself cites them.
- **Two as-built wording precisions vs the ledger/STATE (reported, then recorded this close-out):** (a) ledger #13 said "AudioDeviceCallback in a singleton" — as built, the callback registers in `LocationService` and *feeds* the GPS-monitor-shaped `ExternalSpeakerMonitor` singleton (ledger #13 wording refined in STATE.md); (b) there is no `DRIVER_HAIL_RIDE` event *type* — the rows are `HAIL_AND_RIDE_SECTION_STARTED`/`_ENDED` carrying `trigger_method = DRIVER_HAIL_RIDE` (STATE item 5 wording corrected, so the glass pass checks the right column).
- **One-off push exception (owner-directed, this session only):** Claude pushed pds-android `cdb4f3e`. **Not a standing change** — the code-repo human-review-and-push gate remains in force unchanged.

**Verified:** Every new CLAUDE.md claim verified against code via an Explore pass (file paths, mechanism ordering, state-machine states, event/trigger_method naming; HEAD `eedeb4f`, clean tree, confirmed pre-commit). Docs-only — no build, no tests to run, no hardware involvement; nothing touched the compliance surface.

**What's next:** STATE item 5 — the consolidated GPS glass pass on the Lenovo tablet (route `575261eb` + Lockito), unchanged. The "after item 5" docs touch is now already done, so item 5 closes cleanly on its own when verified.

**Banked / open:** (carried forward; the Android CLAUDE.md docs-touch item is cleared)
- ANDROID-3/4 Sentry issues still open (real July 1 tablet errors) — worth a look before the pilot.
- Pre-existing lint error `scripts/render-fixed-announcements.ts:172` — still banked.
- Render-worker latent gap (unchecked returned-`error` fields) — still banked.
- Three known-stale dashboard text sites (`src/proxy.ts` gating comment, routes empty-state, `triggerReRender` comment) — still banked.
- Deployed `pair-device` one-comment divergence from repo HEAD — accepted, until the next function deploy.
