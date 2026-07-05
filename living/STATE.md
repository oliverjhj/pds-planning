# STATE.md
# Passenger Display System (PDS) — Current State

**This is the living snapshot of the project. It is the current authority for what is built, what remains, and what is being worked on. It is overwritten at the end of each working session to always reflect reality.** Session-by-session history is append-only in `pds-planning/sessions/`. The narrative history up to the July 2026 workflow migration is in `pds-planning/historical/PROJECT-HISTORY.md`. The original planning documents (frozen, historical) are alongside it in `pds-planning/historical/`.

**Last updated:** July 2026 (post CLAUDE.md-rewrite session, 2026-07-05).

---

## What PDS Is

A legally-regulated (UK PSVAR 2023), two-surface product: an **Android tablet app** (`pds-android`) that gives compliant audio + visual passenger information on rail-replacement buses, and a **Next.js dashboard** (`pds-dashboard`) where operators author routes and manage their tablet fleet. Both share a **Supabase** backend; audio is pre-rendered server-side via Google Cloud TTS. Pilot customer: a small bus company. See `PROJECT-HISTORY.md` for how it was built.

---

## Current State — What Is Built

The product is **functionally near-complete on both surfaces.** Everything below is built and hardware-verified unless noted.

**Dashboard (`pds-dashboard`) — Stage 1 complete, deployed to production.** Operator signup/approval, route builder with full-UK NaPTAN stop search, per-stop proximity radius and segment-type (scheduled / hail-and-ride), return-route generation, route list with render-status, fleet/device view, and the per-device journey-summary drill-down. The server-side audio render pipeline (render-then-notify, version-keyed storage, differential re-render) is live.

**Backend (Supabase).** Two projects in lockstep (staging + production). Operator-scoped RLS, the pairing/recovery/pairing-code Edge Functions, the pg_boss render pipeline, Storage for route audio, NaPTAN full-text search, and the `journey_summaries` table. Migrations are believed to be in lockstep on both environments through the latest (a one-shot production audit is queued below to confirm).

**Android (`pds-android`) — feature-complete bar one slice.** Pairing and encrypted credential storage; reactive JWT recovery and lifecycle heartbeat; the full sync engine (route/stop/audio download, three triggers, hands-free dashboard→tablet propagation); GPS stop detection; all automatic announcements (route, next-stop, termination, hail-and-ride) with the alert chime, co-equal screen flash, and text overlay; screen calibration and physically-measured ≥22mm text; the tube-map progress view; journey completion/termination; hail-and-ride sections; diversions (author/observe, GPS auto-skip, replay-on-resume); Level 1 kiosk (screen pinning + default launcher); admin PIN and the full admin menu; and post-journey summary upload.

**Current `main`:** `pds-android` is at `2489345` (frozen-docs removal; before it `7254e78` CLAUDE.md rewrite, then the `835efd8` local-hygiene commit). `pds-dashboard` is at `d0a0cbd` — Stage-1-complete + migrations through 024, deployed to production. Both repo `CLAUDE.md` files are lean and code-grounded (last verified against code July 2026); the frozen v3.9 planning docs are single-homed in `pds-planning/historical/` (refreshed from stale v3.7 in `2af75c4`); the repo `docs/` folders and the dashboard's Next-generated `AGENTS.md` are deleted.

---

## What's Left To Do

In order. Items 2–5 are the pre-pilot set; item 6 follows. (Item 1 — the repo CLAUDE.md rewrite — completed 2026-07-05; numbering is kept stable because the repo CLAUDE.md files cite items 3 and 4 by number.)

### 2. Driver-controls / audio-output slice — 3 commits

The one remaining feature slice. Confirmed unbuilt by code recon; the pilot's **Bluetooth-likely audio** and **hail-and-ride routes** make it compliance-relevant, not polish. All design forks are **already decided** — see the Decision Ledger below; the build should not re-litigate them. Build order (grouped by risk, lowest first; each commit compiles clean):

- **Commit 1 — audio-output hardening (FR-AT-29/30/31).** AudioFocus (`GAIN_TRANSIENT_MAY_DUCK`, per-announcement, play-on-denial for regulated audio); Bluetooth-disconnect handling (auto-fallback to built-in speaker + persistent driver warning mirroring the amber GPS-loss marker); and the keep-alive (the bundled `silent_keepalive.wav`, currently unreferenced dead weight, played every ~10 min during a journey). All additive; touches no schema and no detection core.
- **Commit 2 — volume + Emergency Mute (FR-AT-44).** Driver-panel volume slider bound to live stream volume clamped ≥ floor; hardware volume-button interception; and the Emergency Mute toggle hoisted into the always-visible panel header. Logs mute engage/release to `journey_events`. No schema bump (mute is in-memory — see ledger).
- **Commit 3 — manual hail-and-ride fallback (FR-AT-26/41) — lands last and alone.** A driver-panel control to manually fire H&R section start/end when GPS misdetects. This is the only part that touches the **verified detection core** (it introduces a runtime "manual H&R active" state the next-stop-suppression check must consult), so it is isolated to its own commit with dedicated interpreter unit tests and the full-app GPS glass pass, exactly as the diversion auto-skip slice was isolated.

### 3. Edge Function Sentry

The 8 Supabase Edge Functions currently have **zero error telemetry** (Sentry is wired on Android and the Next.js dashboard only). This is the sole server-side visibility for the pilot — a silent render/pairing failure is presently invisible. Wire Sentry into the functions before pilot. Low effort, high diagnostic payoff.

### 4. Production-shape audit (one-shot, read-only, assume-nothing)

Confirm the deployed production state matches what the record claims: that migrations 020–024 and `pair-device` v8 are genuinely deployed on production (resolving a historical staging-vs-prod contradiction), and resolve whether a stale "deferred" comment in the deployed `pair-device` (around `index.ts:177`) reflects a real deployed-version lag or is just stale text. The staging-pinned MCP cannot read production; use the Supabase CLI or dashboard migration history against the production project, or temporarily re-point the MCP at production.

### 5. Full-app GPS glass pass (verification, not building)

One consolidated hardware run on the Lenovo tablet driving route `575261eb` ("Marford to Guildhall") with Lockito: verify the four deferred diversion-skip cases (plain mid-route skip, H&R-silent landing, manual-override, empty-set — all unit-green, not yet glass-verified), the new manual-H&R fallback from commit 3, and an eyeball of the tube-map hail-and-ride rendering (diamonds + dashed run).

### 6. Then: Stage 4 + pilot

Firebase staging/production project split (currently a single shared Firebase project), then the pilot with the bus company.

---

## Defer Without Guilt (post-pilot)

- **Repeat-last-announcement button** (FR-AT-41) — a Should-Have; needs a new retained-last-announcement holder that doesn't exist today. Genuine work, low pilot value.
- **The ~9 server-alignment hardening items** (pair-device `Retry-After`, RLS `WITH CHECK` tightening, recover-device rate-limit alignment, FCM payload naming, auto-deregister compound condition, etc.) — most are defence-in-depth for a multi-operator fleet at scale, not one-bus-pilot risk. **Triage, don't clear** — fix only any that a pilot driver/passenger could actually experience.
- **The frozen-doc "divergences"** — under the new "docs are evidence, not authority" framing, divergence between the frozen planning docs and the current implementation is now *expected and fine*. The old plan to run a re-planning campaign to reconcile them is **dropped**; no action needed. (The divergences are recorded in code comments where they matter.)

## Bank and Assess (narrow, non-blocking)

- `journey_skipped_stops` not cleared on stale-recovery auto-clear (`RunLaunchRecoveryUseCase`) — a narrow orphaned-rows edge, largely masked because the journey-start path clears the skip set anyway. Assess when convenient; not pilot-blocking.
- `devices.active_route_id` sync stage is a reserved no-op — server-alignment domain, deferred.
- Stationary-detection timeout is hardcoded (could be admin-configurable).
- The Sentry PII-scrubber is a stub (`SentryPiiScrubber`) — real scrubbing logic pending (relevant when Edge Function Sentry lands, item 3).
- Four known-stale dashboard text sites, catalogued in the dashboard CLAUDE.md's "known-stale text" gotcha: the "route builder coming in the next release" empty-state (`routes/page.tsx:59` — the builder exists); the misleading `triggerReRender` comment (`lib/actions/routes.ts:372–376` — calls the real enqueue path a "stub"); the `src/proxy.ts:~10` comment claiming gating lives in the dashboard layout (it lives in the middleware — Rule 3); and the `pair-device/index.ts:177` "deferred unique index" comment (repo-side stale — migration 024 landed the index; whether the *deployed* function matches is item 4's question). Cosmetic; sweep up opportunistically.

---

## Decision Ledger (standing decisions in force)

These are the decisions that currently shape how work is done on this codebase. Full historical decision detail is in `PROJECT-HISTORY.md` and the code itself; this is the curated set still load-bearing.

**Compliance rules (inviolable; every announcement-touching change inherits them):**
1. **Co-equal visual** — the screen flash and text overlay for a regulated announcement fire regardless of `audio_enabled`; a silent tablet still shows them. Any audio-mute must gate *only* the audio, never the visual.
2. **Asymmetric undroppable locking** — a regulated announcement interrupts a non-regulated one; a non-regulated one never interrupts; regulated-vs-regulated drops the later. A legally-required announcement is never silently lost.
3. **Event-index vs approached-index discipline** — the announced/passed stop and the approached stop are distinct indices; audio and boundary logic key on *approached*, event logging on *announced*. Terminus emits null, never clamp-to-last.
4. **≥22mm physical text** — passenger text is sized by physical measurement (calibrated) and never truncates below the floor; it grows line count instead.

**Working disciplines:**
5. **Deployed-shape audit before touching any deployed surface** — audit the actual deployed function/RPC/schema shape, don't trust the spec. Has caught real defects repeatedly.
6. **Plan-mode-first** — for any build, produce a plan and get it reviewed before editing; plan-mode review is a real gate, not a rubber stamp.
7. **One-prompt-one-commit; commit via message-file** (not a here-string — avoids stray `@`). Compliance features are glass-verified on the real Lenovo tablet before commit.
8. **Docs are evidence, not authority** — the frozen planning docs record what was designed/built; current implementation + judgment are authoritative where they differ. Divergence is expected.

**Decided design for the driver-controls / audio-output slice (do not re-litigate):**
9. **Whole slice stays schema-free.** No Room migration. Emergency Mute state is an **in-memory journey-scoped singleton** (evaporates on process death — accepted; avoids a schema bump during the workflow migration); it still logs `MUTE_ENGAGED`/`MUTE_RELEASED` to `journey_events` (additive, no migration).
10. **Volume slider** binds to the live accessibility stream, clamped to `floor..100%` via a shared clamp helper; **hardware volume keys** use an explicit key override (not `volumeControlStream`) routed through the same clamp, so the floor is honest at key-time. The admin-menu volume-floor setting stays the PIN-owned authority; the driver slider moves live volume within it.
11. **Emergency Mute** gates inside the announcement player *only* (alongside the existing `audio_enabled` early-return, before the volume-floor raise) — this is architecturally identical to `audio_enabled = false`, which already proves the visual path is untouched (ledger #1). Never gate mute upstream at the announcement coordinator.
12. **AudioFocus** is requested per chimed announcement pair (not per-journey), abandoned in the same release path; a focus *denial never silences a regulated announcement* (request-but-play-regardless, breadcrumb the denial).
13. **Bluetooth disconnect** uses `AudioDeviceCallback` in a singleton shaped like the GPS-signal monitor, registered for the journey's lifetime, with `NEVER_CONNECTED` / `CONNECTED` / `DISCONNECTED` states — only `DISCONNECTED` warns. The warning reuses the two-surface amber-marker pipe (status strip + panel-header re-surface); two simultaneous markers stack vertically.
14. **Keep-alive** plays on a `LocationService` coroutine ticker (not WorkManager — its 15-min floor can't do a 10-min tick), via a **separate silent playback path that never touches the announcement mutex** (so it can never drop or be dropped by a real announcement), skipping the volume-floor and AudioFocus; gated on `audio_enabled`; keeps playing under Emergency Mute (mute is about audible output); runs unconditionally rather than gating on Bluetooth state.
15. **Manual H&R fallback** introduces an **in-memory** `manualHailRideActive` state (not persisted — its correct lifetime is "until the driver ends it or GPS recovers," and the compliance-safe failure direction on a crash is *toward* re-enabling normal announcements) that the next-stop-suppression check ORs with `segment_type` at both call sites. Manual state is authoritative while active: a GPS-derived section start/end is a no-op while it's on. The manual use case flips the state and logs the event even if the announcement itself is drop-overlapped (the state change is the load-bearing effect); the button shows brief press confirmation. Wired through the view-model via dedicated use cases, never the composable touching the announcement bus directly.

**Documentation discipline (added 2026-07-05):**
16. **Repo CLAUDE.md rule numbers are frozen.** Code comments in both repos cite the architectural rules by number ("Rule N"), so rule numbers are never reused, reordered, or repurposed — a retired rule's number stays retired, and new rules append at the end of the list. New feature areas are documented as conventions unless they carry a genuinely new inviolable constraint.

---

## Key Operational Anchors (current as of this snapshot)

- **Hardware:** Lenovo TB373FU tablet, Android 16, USB serial `HA2DG5NK`. GPS testing via **Lockito** (mock-location app); accuracy must be set well under a stop's proximity radius or fixes are discarded.
- **Primary GPS/H&R test route:** `575261eb-def8-486e-b4ab-320e07a3ff17` ("Marford to Guildhall", 6 stops, scheduled/H&R/scheduled layout, 80m radii). A separate all-scheduled route and an older H&R route also exist.
- **Canonical staging entities:** one operator (`360ffc45-…`) and one device (`32ab0216-…`).
- **Supabase:** staging ref `febcpudiorwnsbcphcng`, production ref `qopkjsihmdplmsoqrzji`. Keep the MCP read-only and staging-pinned at rest.
- **Local repo paths:** `C:\Users\dev\Documents\GitHub\pds-android` and `…\pds-dashboard`.
