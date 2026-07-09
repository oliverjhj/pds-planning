# STATE.md
# Passenger Display System (PDS) — Current State

**This is the living snapshot of the project. It is the current authority for what is built, what remains, and what is being worked on. It is overwritten at the end of each working session to always reflect reality.** Session-by-session history is append-only in `pds-planning/sessions/`. The narrative history up to the July 2026 workflow migration is in `pds-planning/historical/PROJECT-HISTORY.md`. The original planning documents (frozen, historical) are alongside it in `pds-planning/historical/`.

**Last updated:** July 2026 (post edge-function-Sentry session, 2026-07-09).

---

## What PDS Is

A legally-regulated (UK PSVAR 2023), two-surface product: an **Android tablet app** (`pds-android`) that gives compliant audio + visual passenger information on rail-replacement buses, and a **Next.js dashboard** (`pds-dashboard`) where operators author routes and manage their tablet fleet. Both share a **Supabase** backend; audio is pre-rendered server-side via Google Cloud TTS. Pilot customer: a small bus company. See `PROJECT-HISTORY.md` for how it was built.

---

## Current State — What Is Built

The product is **functionally near-complete on both surfaces.** Everything below is built and hardware-verified unless noted.

**Dashboard (`pds-dashboard`) — Stage 1 complete, deployed to production.** Operator signup/approval, route builder with full-UK NaPTAN stop search, per-stop proximity radius and segment-type (scheduled / hail-and-ride), return-route generation, route list with render-status, fleet/device view, and the per-device journey-summary drill-down. The server-side audio render pipeline (render-then-notify, version-keyed storage, differential re-render) is live. The dashboard's Sentry is **actually live as of 2026-07-09** — the code had been wired since Stage 1 but no Sentry project/DSN existed behind it, so it had been silently no-oping in production (session finding); now provisioned with an `environment` tag and zeroed sample rates.

**Backend (Supabase).** Two projects in lockstep (staging + production). Operator-scoped RLS, the pairing/recovery/pairing-code Edge Functions, the pg_boss render pipeline, Storage for route audio, NaPTAN full-text search, and the `journey_summaries` table. **All 8 Edge Functions carry Sentry error telemetry (item 3, completed and verified on staging AND production 2026-07-09)** via the shared `supabase/functions/_shared/` modules — error-only, quota-disciplined, with a pure vitest-tested NFR-R-07 scrubber (the repo's first test suite). All 8 deployed functions match repo HEAD on both environments as of 2026-07-09 (freshly deployed this session). Migrations are believed to be in lockstep on both environments through the latest (the one-shot production audit, item 4, now covers only the migrations side).

**Android (`pds-android`) — feature-complete.** Pairing and encrypted credential storage; reactive JWT recovery and lifecycle heartbeat; the full sync engine (route/stop/audio download, three triggers, hands-free dashboard→tablet propagation); GPS stop detection; all automatic announcements (route, next-stop, termination, hail-and-ride) with the alert chime, co-equal screen flash, and text overlay; screen calibration and physically-measured ≥22mm text; the tube-map progress view; journey completion/termination; hail-and-ride sections; diversions (author/observe, GPS auto-skip, replay-on-resume); Level 1 kiosk (screen pinning + default launcher); admin PIN and the full admin menu; and post-journey summary upload. The driver-controls/audio-output slice landed 2026-07-08: per-announcement AudioFocus, external-speaker disconnect monitoring (two-surface amber warning + driver dismiss, FR-AT-63 device events), the silent Bluetooth keep-alive, the driver volume slider + app-wide hardware volume-key clamp against the PIN-owned floor, Emergency Mute (audio-only, third amber marker), and the manual hail-and-ride fallback buttons (unit-verified; glass folded into item 5).

**Current `main`:** `pds-android` is at `1585a47` (unchanged this session — the driver-controls/audio-output slice). `pds-dashboard` is at `9744643` — the edge-function-Sentry slice, nine commits (`bf046cd` shared telemetry+scrubber, `18cbaec` HTTP functions, `f18c503` scheduled workers, `41e6c3b` temp verification hook, `638e9d1` dashboard config fix, `58b7436` hex-threshold fix, `14689f2` capture-context fix, `5359d46` hook strip, `9744643` docs), all deployed to staging and production. Both repo `CLAUDE.md` files are lean and code-grounded; the frozen v3.9 planning docs are single-homed in `pds-planning/historical/`.

---

## What's Left To Do

In order. Items 4–5 are the pre-pilot set; item 6 follows. (Item 1 — repo CLAUDE.md rewrite — completed 2026-07-05. Item 2 — driver-controls/audio-output slice — completed 2026-07-08. Item 3 — Edge Function Sentry + dashboard DSN fix — completed 2026-07-09, verified on both environments. Numbering is kept stable because the repo CLAUDE.md files cite items 3 and 4 by number.)

### 4. Production-shape audit (one-shot, read-only, assume-nothing) — SCOPE SHRUNK 2026-07-09

Confirm the deployed production state matches what the record claims. **The Edge Function half is resolved by construction**: the 2026-07-09 session deployed all 8 functions to production from repo HEAD, so deployed `pair-device` now matches the repo (and carries migration 024's semantics). What remains: (a) confirm **migrations 020–024** are genuinely applied on production, and (b) fix the stale repo-side comment in `pair-device/index.ts` (~177–180, claims the android_id unique index is "deferred" — 024 landed it). The staging-pinned MCP cannot read production; use the Supabase CLI (now logged in on this machine) or the sanctioned temporary MCP re-point at production.

### 5. Full-app GPS glass pass (verification, not building)

One consolidated hardware run on the Lenovo tablet driving route `575261eb` ("Marford to Guildhall") with Lockito: verify the four deferred diversion-skip cases (plain mid-route skip, H&R-silent landing, manual-override, empty-set — all unit-green, not yet glass-verified); the manual-H&R fallback (built 2026-07-08, unit-green): manual start suppresses next-stop + segment-derived boundaries, always-fire non-transition end press announces + logs, drop-overlap press shows the on-button confirmation, `DRIVER_HAIL_RIDE` event rows land, and the second "leaving" announcement after a manual END inside a physical H&R region is *expected* (documented in `TriggerManualHailRideUseCase`); and an eyeball of the tube-map hail-and-ride rendering (diamonds + dashed run).

### 6. Then: Stage 4 + pilot

Firebase staging/production project split (currently a single shared Firebase project), then the pilot with the bus company.

---

## Defer Without Guilt (post-pilot)

- **Repeat-last-announcement button** (FR-AT-41) — a Should-Have; needs a new retained-last-announcement holder that doesn't exist today. Genuine work, low pilot value.
- **The ~9 server-alignment hardening items** (pair-device `Retry-After`, RLS `WITH CHECK` tightening, recover-device rate-limit alignment, FCM payload naming, auto-deregister compound condition, etc.) — most are defence-in-depth for a multi-operator fleet at scale, not one-bus-pilot risk. **Triage, don't clear** — fix only any that a pilot driver/passenger could actually experience.
- **The frozen-doc "divergences"** — under the "docs are evidence, not authority" framing, divergence between the frozen planning docs and the current implementation is *expected and fine*. The old re-planning campaign is **dropped**. (New instance this session: the six-DSN Sentry plan was superseded by one-project-per-surface + environment tag — recorded in the dashboard CLAUDE.md/README, ledger #20.)
- **@sentry/deno v10 upgrade** — pinned at 8.55.2 (the Supabase-docs-validated major); bumping is a deliberate toolchain task, not feature collateral.

## Bank and Assess (narrow, non-blocking)

- `journey_skipped_stops` not cleared on stale-recovery auto-clear (`RunLaunchRecoveryUseCase`) — a narrow orphaned-rows edge, largely masked because the journey-start path clears the skip set anyway. Assess when convenient; not pilot-blocking.
- `devices.active_route_id` sync stage is a reserved no-op — server-alignment domain, deferred.
- Stationary-detection timeout is hardcoded (could be admin-configurable).
- **The Android Sentry PII-scrubber is still a stub** (`SentryPiiScrubber` — pass-through with the contract in its header). The policy is now *implemented* on the edge surface (`_shared/scrub.ts`, where the real secrets flow); the Android implementation remains banked. Dashboard beforeSend scrubbing deliberately out of scope (Next SDK attaches no request bodies by default).
- **Both repo CLAUDE.mds' "Available MCP Tools" sections are stale** — they say four user-scope MCP servers; there are now six (Sentry + Vercel added 2026-07-09). Update on the next deliberate docs touch.
- Pre-existing lint error `scripts/render-fixed-announcements.ts:172` (prefer-const) fails `npm run lint` in pds-dashboard — untouched per no-drive-by; sweep opportunistically.
- Render-worker latent gap (noted 2026-07-09, not fixed — scope): the returned-`error` fields of `pgboss_complete_render_job` and the routes-status UPDATE are unchecked; only thrown exceptions reach the Sentry capture sites.
- Sentry ops: owner to eventually confirm spike protection / a volume alert on the org (a DB outage would make the render worker's claim capture fire ~1/min); FCM-secret-missing is deliberately log-only. Two real Android issues (ANDROID-3/4, from July 1 tablet testing) sit unresolved in Sentry — worth a look sometime.
- Known-stale dashboard text sites remaining: the "route builder coming in the next release" empty-state (`routes/page.tsx:59`); the misleading `triggerReRender` comment (`lib/actions/routes.ts:372–376`); the `src/proxy.ts:~10` gating comment; and the `pair-device/index.ts:~177` "deferred unique index" comment (now item 4's part (b)). Cosmetic; sweep deliberately.

---

## Decision Ledger (standing decisions in force)

These are the decisions that currently shape how work is done on this codebase. Full historical decision detail is in `PROJECT-HISTORY.md` and the code itself; this is the curated set still load-bearing.

**Compliance rules (inviolable; every announcement-touching change inherits them):**
1. **Co-equal visual** — the screen flash and text overlay for a regulated announcement fire regardless of `audio_enabled`; a silent tablet still shows them. Any audio-mute must gate *only* the audio, never the visual.
2. **Asymmetric undroppable locking** — a regulated announcement interrupts a non-regulated one; a non-regulated one never interrupts; regulated-vs-regulated drops the later. A legally-required announcement is never silently lost.
3. **Event-index vs approached-index discipline** — the announced/passed stop and the approached stop are distinct indices; audio and boundary logic key on *approached*, event logging on *announced*. Terminus emits null, never clamp-to-last.
4. **≥22mm physical text** — passenger text is sized by physical measurement (calibrated) and never truncates below the floor; it grows line count instead.

**Working disciplines:**
5. **Deployed-shape audit before touching any deployed surface** — audit the actual deployed function/RPC/schema shape, don't trust the spec. Has caught real defects repeatedly (latest: 2026-07-09 — `generate-pairing-code`/`enqueue-render-job` are `verify_jwt: true` unlike the other six; a blanket `--no-verify-jwt` redeploy would have silently changed their auth posture).
6. **Plan-mode-first** — for any build, produce a plan and get it reviewed before editing; plan-mode review is a real gate, not a rubber stamp.
7. **One-prompt-one-commit; commit via message-file** (not a here-string — avoids stray `@`). Compliance features are glass-verified on the real Lenovo tablet before commit.
8. **Docs are evidence, not authority** — the frozen planning docs record what was designed/built; current implementation + judgment are authoritative where they differ. Divergence is expected.

**As-built design record for the driver-controls / audio-output slice (built 2026-07-08; #9–15 were the pre-build decisions, all honoured — see also #17–19 for the plan-review refinements):**
9. **Whole slice stays schema-free.** No Room migration. Emergency Mute state is an **in-memory journey-scoped singleton** (evaporates on process death — accepted; avoids a schema bump during the workflow migration); it still logs `MUTE_ENGAGED`/`MUTE_RELEASED` to `journey_events` (additive, no migration).
10. **Volume slider** binds to the live accessibility stream, clamped to `floor..100%` via a shared clamp helper; **hardware volume keys** use an explicit key override (not `volumeControlStream`) routed through the same clamp, so the floor is honest at key-time. The admin-menu volume-floor setting stays the PIN-owned authority; the driver slider moves live volume within it.
11. **Emergency Mute** gates inside the announcement player *only* (alongside the existing `audio_enabled` early-return, before the volume-floor raise) — this is architecturally identical to `audio_enabled = false`, which already proves the visual path is untouched (ledger #1). Never gate mute upstream at the announcement coordinator.
12. **AudioFocus** is requested per chimed announcement pair (not per-journey), abandoned in the same release path; a focus *denial never silences a regulated announcement* (request-but-play-regardless, breadcrumb the denial).
13. **Bluetooth disconnect** uses `AudioDeviceCallback` in a singleton shaped like the GPS-signal monitor, registered for the journey's lifetime, with `NEVER_CONNECTED` / `CONNECTED` / `DISCONNECTED` states — only `DISCONNECTED` warns. The warning reuses the two-surface amber-marker pipe (status strip + panel-header re-surface); two simultaneous markers stack vertically.
14. **Keep-alive** plays on a `LocationService` coroutine ticker (not WorkManager — its 15-min floor can't do a 10-min tick), via a **separate silent playback path that never touches the announcement mutex** (so it can never drop or be dropped by a real announcement), skipping the volume-floor and AudioFocus; gated on `audio_enabled`; keeps playing under Emergency Mute (mute is about audible output); runs unconditionally rather than gating on Bluetooth state.
15. **Manual H&R fallback** introduces an **in-memory** `manualHailRideActive` state (not persisted — its correct lifetime is "until the driver ends it or GPS recovers," and the compliance-safe failure direction on a crash is *toward* re-enabling normal announcements) that the next-stop-suppression check ORs with `segment_type` at both call sites. Manual state is authoritative while active: a GPS-derived section start/end is a no-op while it's on. The manual use case flips the state and logs the event even if the announcement itself is drop-overlapped (the state change is the load-bearing effect); the button shows brief press confirmation. Wired through the view-model via dedicated use cases, never the composable touching the announcement bus directly.

**Documentation discipline (added 2026-07-05):**
16. **Repo CLAUDE.md rule numbers are frozen.** Code comments in both repos cite the architectural rules by number ("Rule N"), so rule numbers are never reused, reordered, or repurposed — a retired rule's number stays retired, and new rules append at the end of the list. New feature areas are documented as conventions unless they carry a genuinely new inviolable constraint.

**Plan-review refinements from the driver-controls slice (added 2026-07-08; each owner-approved in plan review):**
17. **Manual H&R buttons are always-fire** (PRD FR-AT-41 verbatim, chosen over a strict transition-only toggle): every press logs the section event, fires the chimed announcement "regardless of segment boundaries", and idempotently sets/clears the manual override — a non-transition press still announces, giving one-tap recovery when GPS auto-fired a section start but missed the end. A repeated announcement from a re-press is deliberate correction behaviour, **not a bug to fix**.
18. **Focus-denial asymmetry:** a non-regulated announcement (next-stop / route-and-destination) **drops** on an AudioFocus denial (breadcrumbed); only regulated audio plays regardless (#12). The keep-alive requests no focus at all (#14).
19. **The speaker-disconnect warning is driver-dismissible** (PRD FR-AT-30 verbatim, chosen over the plain GPS-marker mirror) — a per-episode `DISMISSED` machine state, with the dismiss affordance in the panel header only; and an engaged Emergency Mute shows a third amber "Audio muted" strip marker, so a silently-muted tablet is never visually normal.

**Sentry telemetry posture (added 2026-07-09; owner-approved in plan review):**
20. **One Sentry project per surface** (`android` / `dashboard` / `edge-functions`, org `passenger-display-system`, EU region), staging vs production split by the `environment` tag — supersedes the frozen six-DSN plan (Android precedent). **Sample rates are 0 on every surface** (errors and crashes only; protects the shared 5k/month free tier). Edge environment is derived from the `SUPABASE_URL` project ref (override: `SENTRY_ENVIRONMENT`); the edge DSN is the `SENTRY_DSN_EDGE` Supabase secret (same value both projects); the dashboard DSN is `NEXT_PUBLIC_SENTRY_DSN_DASHBOARD` in Vercel. SDK pinned `npm:@sentry/deno@8.55.2`.
21. **Edge capture rules (empirically forced, regression-guarded):** the scrubber's hex-redaction threshold is **40, never 32** — Sentry's own protocol IDs are exactly 32 hex and a ≥32 rule makes ingest silently 400-reject every event; and per-event context goes through **captureException's capture-context argument, never `withScope`** — scope mutation does not isolate on the per_worker edge runtime (contexts provably leaked across captures). Quota discipline: expected outcomes (401/403/404/429) are never captured; `audio-cleanup-worker` emits one end-of-run summary event, not per-route events; dedupe integration on.

---

## Key Operational Anchors (current as of this snapshot)

- **Hardware:** Lenovo TB373FU tablet, Android 16, USB serial `HA2DG5NK`. GPS testing via **Lockito** (mock-location app); accuracy must be set well under a stop's proximity radius or fixes are discarded.
- **Primary GPS/H&R test route:** `575261eb-def8-486e-b4ab-320e07a3ff17` ("Marford to Guildhall", 6 stops, scheduled/H&R/scheduled layout, 80m radii). A separate all-scheduled route and an older H&R route also exist.
- **Canonical staging entities:** one operator (`360ffc45-…`) and one device (`32ab0216-…`).
- **Supabase:** staging ref `febcpudiorwnsbcphcng`, production ref `qopkjsihmdplmsoqrzji`. Keep the MCP read-only and staging-pinned at rest. The Supabase CLI is logged in on this machine (read-only checks like `secrets list` / `functions list` available; `WORKER_SHARED_SECRET` is the same value on both projects).
- **Sentry:** org `passenger-display-system` (EU region, `de.sentry.io` ingest); projects `android` / `dashboard` / `edge-functions`, one DSN each, environment-tagged (ledger #20). Sentry MCP connected at user scope (reads + issue updates work; project creation is owner-only). Vercel MCP also connected (deployments/logs; env vars remain dashboard-only).
- **Local repo paths:** `C:\Users\dev\Documents\GitHub\pds-android` and `…\pds-dashboard`.
