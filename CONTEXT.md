# CONTEXT.md
# Passenger Display System (PDS) — Living Context

**Version:** v3.8
**Last Updated:** May 2026

This document is the architect's living memory across chats. It is the only knowledge file that changes during the project's lifetime; the others (PRD, Data-Architecture, Compliance, WORKFLOW) are frozen.

Re-uploaded to the Claude Project whenever it's updated. Read at the start of every new architect chat.

## Update notes

**v3.8 (May 2026)** — Round-2 close-out. Resolved parked contradictions (decisions #17, #20, #21, #26 rewritten; #28 and #29 updated to drop hard-delete language and move FR-WD-12 divergence to structural hash). Appended decisions #31–#51 covering round-2 changes. Added a round-2 session-history entry. Refreshed reference notes. All five planning documents now at v3.8 alongside both CLAUDE.md files; the frozen-documents rule re-engages with the new WORKFLOW §13.7 CLAUDE.md-propagation requirement in force.

---

## Orientation

The Passenger Display System (PDS) is a two-surface SaaS product providing legally compliant audio and visual passenger information on UK buses operating rail replacement services. The product consists of an Android tablet application (kiosk-mode, offline-first, GPS-driven, regulation-compliant) and a Next.js web dashboard where bus operators author routes and manage their fleet. Both share a single Supabase backend.

An earlier build attempt at this product was abandoned due to scope drift and architectural disconnects between planning and implementation. This project is a clean restart with revised planning documents and tighter workflow conventions (see WORKFLOW.md).

---

## 1. Current State

### What Exists

- Five planning documents finalised at v3.8 following two post-adversarial-review re-planning campaigns: a seven-task round 1 (v3.0 → v3.7) and a four-task round 2 (v3.7 → v3.8):
  - PRD.md v3.8
  - Data-Architecture.md v3.8
  - Compliance-Mapping-Matrix.md v3.8
  - WORKFLOW.md v3.8
  - CLAUDE.md (Android) v3.8 and CLAUDE.md (Dashboard) v3.8 — two separate files, one per repository, brought into the versioning system at v3.8 in the round-2 close-out
- No code yet
- Repositories not yet created (Android app and dashboard will live in separate repos)
- Supabase project not yet created
- No claude-code chats yet

### Build Structure Note

The build is staged for tractability (Stage 1: dashboard MVP; Stage 2: Android foundation; Stage 3: Android journey runtime; Stage 4: compliance polish and pilot) but the release is single and coherent. There is no "v1" and "v2" split, no phased rollout to operators, and no partial product shipped to customers. The four stages are an internal sequencing tool for the development workflow. Operators receive the full product — both surfaces, all compliance features — in one release. The MoSCoW list and all PRD requirements apply to that single release.

### What's Next

**Stage 1: Web Dashboard MVP** (Next.js + TypeScript + Supabase Auth + shadcn/ui).

The dashboard must exist before the Android app because tablets need pairing codes to register, and the dashboard is what generates them.

Stage 1 scope:
- Next.js project scaffolding
- Supabase project setup with schema migrations
- Operator signup, login, password reset flows
- Manual approval workflow with email notification to the system administrator
- Routes CRUD with NaPTAN search
- Device pairing code generation
- Device fleet view with online/offline status (including per-device audio-enabled toggle)
- Edge Functions for `generate-pairing-code`, `pair-device`, `recover-device`

Subsequent stages (handled in later chats):
- Stage 2: Android app foundation (pairing, Room, sync, route browsing, kiosk mode)
- Stage 3: Android journey runtime (GPS service, stop detection, stylised view, audio file playback, driver controls)
- Stage 4: Compliance polish and pilot deployment

---

## 2. Decisions Log

Architectural decisions, in order made. Append only — don't delete past entries.

1. Restart the project rather than salvage the previous codebase. The previous build was too disjointed.
2. One product version, no v1/v2 phases. Build the full intended product directly.
3. Add a web dashboard as a co-equal surface alongside the Android app. Two surfaces, one Supabase backend.
4. Self-signup with manual approval. The system administrator approves operators by setting `status = 'active'` on the operator's row in Supabase. Stripe billing is deferred to a later release; initial billing is manual invoicing.
5. One login per bus company. No team or multi-user accounts in the initial release.
6. Tablet is read-only for routes. Routes are authored exclusively on the dashboard.
7. Stylised tube-map view (custom Android Canvas drawing) for journey progress visualisation, not a Google Maps embed. Eliminates per-load API costs and keeps the GPS load low.
8. Pairing-code device registration replaces the previous operator-code approach. 6-digit codes, 10-minute expiry, single-use.
9. Both surfaces use Supabase Auth uniformly. Operators are regular Supabase Auth users; devices are anonymous Supabase Auth users created by the `pair-device` Edge Function.
10. NaPTAN scope: full UK bus stops + railway stations (~400,000 records). NaPTAN lives exclusively in Supabase (`naptan_stations` table), used by the dashboard route builder. The tablet does not bundle or store NaPTAN data — stop names, coordinates, and NaPTAN IDs travel with routes via `route_stops` (copied at dashboard route-creation time). Custom drop-pin stops deferred. Full UK addressing data is out of scope.
11. Two separate repositories (Android app, web dashboard) with two separate CLAUDE.md files. PRD/Data-Architecture/Compliance Matrix references to "CLAUDE.md" refer to the Android one (where compliance rules like the alert chime are codified).
12. Frozen-documents rule established. PRD, Data-Architecture, Compliance Matrix, and WORKFLOW do not change during the build. Evolving information goes in this CONTEXT.md instead. If a frozen document genuinely needs to change, that triggers a deliberate re-planning session, not a routine amendment.
13. Tech stack for the dashboard: Next.js 15+ with App Router, TypeScript strict, Tailwind CSS, shadcn/ui components, React Hook Form + Zod for validation, server components and server actions as the default data-fetching pattern (no TanStack Query in the initial release).
14. Hosting: Vercel for the dashboard (free tier sufficient for the initial customer set).

The following entries (15–30) were added during the post-adversarial-review re-planning campaign (Tasks 1–7, May 2026):

15. Per-stop proximity radius (set per stop in dashboard, 200m default) replaces the previous global radius setting. Two-stop look-ahead tolerance added: app monitors stops N and N+1; auto-advances past N if N+1 is entered first, logging N as passed without detection. GPS accuracy gate discards fixes worse than the target stop's radius, preventing bad-fix misfires. (Task 1)
16. Hail-and-ride sections are a first-class data concept via `route_stops.segment_type` ('scheduled' / 'hail_and_ride'). GPS state machine auto-fires H&R section start and end announcements at segment boundaries defined in route data. Manual driver buttons retained as fallback. Diversion skip state handled via local `journey_skipped_stops` Room table. (Task 2)
17. Audio architecture: pre-rendered MP3 files. **Round 2 rewrite (see decisions #31–#32):** the synchronous `render-route-audio` Edge Function originally introduced here is replaced by a `pg_boss` job queue. The dashboard calls the `enqueue-render-job` Edge Function on every save; the queued `render-route-audio` `pg_boss` job is consumed by the `audio-render-worker`, which calls Google Cloud TTS (voice `en-GB-Neural2-B`, locked), writes versioned MP3s to the `route-audio` Storage bucket at `{operator_id}/{route_id}/{route_version}/...`, dispatches the post-render FCM push, and uses `audio_announcement_hash` for differential re-render (skip synthesis when text unchanged). On-device TTS remains removed entirely. Fixed announcement texts (termination, H&R start/end, diversion start/end, alert chime, silent keepalive) remain bundled in APK `res/raw/`, rendered once using the same locked voice. Diversion audio remains a generic fixed phrase; specific affected stop names are conveyed visually via tube-map strikethrough. (Task 3 / Round 2 Task 1)
18. Device status enum: `status TEXT CHECK IN ('pending', 'active', 'suspended')` replaces `is_approved BOOLEAN`. Dashboard and tablet UI show distinct messages per state; suspended and pending are never conflated. (Task 4)
19. Device secret for recovery: 256-bit cryptographically random secret generated at pairing, returned to tablet once in the pairing response, stored as SHA-256 hash server-side. `recover-device` requires both Android ID (row lookup key) and device secret (authentication). Android ID alone is not a credential. (Task 4)
20. FCM promoted to Must Have. Three-trigger sync model formalised: ConnectivityManager (offline→online), FCM push notification (responsive, sub-5-minute propagation), 30-min WorkManager periodic (safety net). FCM is the primary mechanism for timely route propagation to online tablets. **Round 2 update:** the post-save FCM push is dispatched by the `audio-render-worker` on render success, not by a routes-table DB trigger or by the dashboard. Render-then-FCM ordering is mandatory — tablets must not be told to sync until the new audio is in Storage, or they will 404 on download. FCM payload is locked to data-only `{ type, operator_id, trigger }` (see decision #47). (Task 5 / Round 2 Tasks 1 & 3)
21. Heartbeat mechanism: 2-minute `last_seen_at` update, independent of route sync. **Round 2 rewrite (see decision #33):** ownership moved to a lifecycle-based `HeartbeatController` driven by `ProcessLifecycleOwner`, not the foreground GPS service. Two paths: app-foregrounded (reliable; ticks whenever any Activity is RESUMED — covers route browsing, route detail, admin menu, and active journeys) and background/idle (best-effort WorkManager PeriodicWorkRequest; not guaranteed on hostile-OEM hardware — documented acceptable gap). The foreground GPS service is no longer the heartbeat owner; the two have independent lifecycles and reliability profiles. Fleet managers should treat idle tablets as "not in service." (Task 5 / Round 2 Task 2)
22. Kiosk Level 2 (Device Owner mode / Android Lock Task Mode) deferred to Could Have. Level 1 soft kiosk (screen pinning + default launcher registration) is the only kiosk mode in the initial release. Architecture accommodates Level 2 without redesign; only provisioning differs. (Task 6)
23. Tablet NaPTAN bundle cut. Tablet holds no NaPTAN data. Stop names, coordinates, and NaPTAN IDs travel with routes via `route_stops` snapshot copied at dashboard route-creation time. NaPTAN is dashboard-only. (Task 6)
24. Upload-sync scaffolding cut. Sync is download-only in the initial release. The "check → upload → download" sequence and all upload-phase language removed from PRD, Data-Architecture, and WORKFLOW. (Task 6)
25. Visual alert defined (FR-AT-65): 500ms high-contrast screen flash fires simultaneously with the audio chime, before the announcement overlay text appears. Same four trigger types as the audio chime (termination, diversion start, H&R start, H&R end). Together with FR-AT-27 (audio chime), constitutes the combined alert required by Regs 8(2), 10(2)(b), 11(2)(b), 11(5)(b). (Task 7)
26. Multi-tablet audio designation: `audio_enabled BOOLEAN` per device in the `devices` table, configurable from the dashboard fleet view. First tablet paired to an operator defaults to `true`; subsequent tablets default to `false`. Tablets with `audio_enabled = false` suppress all audio output AND skip audio downloads entirely (round-2 clarification — see decision #38) while continuing full visual display. **Honest scope (round-2 reframe — see decision #34):** software provides the per-device toggle and a non-blocking dashboard warning when more than one tablet in a fleet has audio enabled. The system **cannot** enforce one-per-bus because it cannot tell which physical bus a tablet is on. Enabling exactly one tablet per bus is **operator responsibility**, not a software-enforced constraint. (Task 7 / Round 2 Tasks 2 & 3)
27. `device_pairing_codes` primary key changed from `code TEXT` to `id UUID DEFAULT gen_random_uuid()`. `code` demoted to a regular UNIQUE indexed column. Decouples row identity from the code value, avoiding entanglement between cleanup, audit retention, and collision checking. (Task 7)
28. `devices.active_route_id` FK changed to `ON DELETE SET NULL`. Retained as a defensive constraint against any future route hard-deletion path. **Round 2 update (see decision #48):** hard-delete language for routes has been removed from the planning suite — routes are not hard-deleted on a fixed schedule in the initial release. The `ON DELETE SET NULL` constraint remains in place as defence in depth; the original 90-day-retention justification no longer applies. (Task 7 / Round 2 Task 3)
29. Return route divergence detection. **Round 2 rewrite (see decision #39):** detection is now structural, via SHA-256 content hash comparison. `routes.stops_content_hash` is computed inside `replace_route_with_stops` from the canonical stop-list serialisation; the linked return route stores `stops_content_hash_at_generation` at generation time. Dashboard warns operator when editing a route whose source `stops_content_hash` differs from the linked return's `stops_content_hash_at_generation`; offers [Re-generate return] or [Keep existing]. The original `last_synced_with_return TIMESTAMPTZ` mechanism is deprecated as the divergence trigger (it fired on every save — warning fatigue); the column is retained as a soft audit signal only. Warning-and-regenerate only, not a diff UI. (Task 7 / Round 2 Task 3)
30. Room migration policy added to WORKFLOW.md §10.3: every Room schema change must include a Migration object or an explicit `fallbackToDestructiveMigration()` with rationale; migration must be tested on a real device with pre-existing data before commit. Prevents the schema-mismatch crash class that terminated the previous build. (Task 7)

The following entries (31–51) were added during the round-2 post-adversarial-review re-planning campaign (Round 2 Tasks 1–4, May 2026):

31. Audio pipeline replaced with `pg_boss` job queue + Google Cloud TTS + version-keyed Storage paths + render-then-FCM ordering. Dashboard calls `enqueue-render-job` Edge Function on save; the queued `render-route-audio` job is consumed by `audio-render-worker` (a separately-deployed runtime, not an Edge Function). Worker writes to `route-audio` Storage bucket at `{operator_id}/{route_id}/{route_version}/...`, then dispatches the post-render FCM data-only push. Differential re-render via `audio_announcement_hash` skips synthesis on unchanged route-announcement text. Supersedes the synchronous `render-route-audio` Edge Function from decision #17. (Round 2 / Task 1)
32. TTS provider locked: Google Cloud TTS, voice `en-GB-Neural2-B`. Voice change is itself a deliberate compliance event (Reg 13(4) frequency-range re-verification required). (Round 2 / Task 1)
33. Heartbeat reframed as lifecycle-based: `HeartbeatController` driven by `ProcessLifecycleOwner`. GPS service is not the heartbeat owner. App-foregrounded path is reliable across all RESUMED activities; background/idle path is best-effort `WorkManager`. Supersedes the Handler-in-GPS-service approach from decision #21. (Round 2 / Task 2)
34. `audio_enabled` enforcement framed honestly: software provides per-device toggle and a non-blocking dashboard warning when more than one tablet in a fleet has audio enabled. One-per-bus is operator responsibility, not system-enforced. The system cannot determine which physical bus a tablet is on. Reframes decision #26. (Round 2 / Task 2)
35. Operator suspension honoured at journey end, not mid-journey. If a sync arrives while a journey is active and the captured operator status is `suspended` or `pending`, the app continues the journey to its natural end and then transitions to the Account Status Screen on the next journey-start attempt. Prevents mid-journey UI lock that would strand passengers. (Round 2 / Task 2)
36. Sentry adopted for crash telemetry across all three surfaces: Android (Android SDK, initialised in `Application.onCreate`), dashboard (Next.js SDK with server, client, and edge runtimes), and Edge Functions (Sentry for Deno/Edge runtime). PII stripped in all three. PRD §NFR-R-07 codifies the requirement. (Round 2 / Task 2)
37. Journey summary upload at journey end via `journey_summaries` (Supabase) + `journey_summaries_pending` (Room outbox). Anonymous count metrics only — no PII, no location traces. `journey_summaries_pending` is the **only** legitimate use of `needs_upload` in the initial release (deliberate, scoped exception to the v3.6 removal of upload-sync scaffolding from `routes` / `route_stops`). Uploaded by a small step in the next sync; cleared on success. (Round 2 / Task 2)
38. Tablets with `audio_enabled = false` skip audio downloads entirely, not just playback. The sync's audio-download step is short-circuited at the top, before iterating routes. Saves storage, bandwidth, and avoids paying for audio files the tablet will never play. Clarifies decision #26. (Round 2 / Task 3)
39. `routes.stops_content_hash` replaces timestamp-based divergence detection for FR-WD-12. SHA-256 of canonical stop-list serialisation (form: `{naptan_id}|{stop_order}` per stop in ascending stop_order, newline-joined, UTF-8, lowercase hex). Computed inside `replace_route_with_stops` RPC. Linked return routes store `stops_content_hash_at_generation`; dashboard divergence warning compares the two hashes. Eliminates the warning fatigue from the old `updated_at > last_synced_with_return` comparison (which fired on every edit). Rewrites decision #29. (Round 2 / Task 3)
40. `journey_state` staleness recovery rules: on app restart, recover only if `last_event_at` is within the last 1 hour AND the journey is no older than 8 hours since start. Outside those bounds, discard the state. If recovery lands inside a diversion segment, replay the diversion-start announcement in full (chime + flash + audio). Prevents stale-state ghost journeys after long power-offs. (Round 2 / Task 3)
41. `journey_events` cleanup at both app startup and journey start (defensive double-cleanup). Belt-and-braces against either path failing. (Round 2 / Task 3)
42. FR-AT-67: Google Mobile Services availability check at first run. Blocks journey starts with a clear warning if GMS unavailable; "continue anyway" override available for development/testing (logged to `journey_events`). Production tablets must be GMS-certified. Surfaces a previously-implicit assumption as a runtime check. (Round 2 / Task 3)
43. FR-AT-04: `recover-device` failures classified as transient (5xx, network, 429 — retain cached credentials, retry with backoff) vs terminal (404, 401, `activation_state = 'inactive'`, operator status not `active` — wipe local credentials, force re-pair). Distinct UI per case. Prevents a transient network blip from dropping an in-service tablet to the pairing screen. (Round 2 / Task 3)
44. `devices.status` renamed to `devices.activation_state` with explicit CHECK constraint. Disambiguates from `operators.status` (the three-state enum from decision #18). Reduces cognitive collision between operator-level and device-level state. (Round 2 / Task 3)
45. 30-day heartbeat-billable / 60-day auto-deregister policy formalised. Tablets that have not heart-beated for 30 days drop out of billable count; tablets that have not heart-beated for 60 days are auto-deregistered (`activation_state` flipped to `inactive`, requires re-pair). Implemented as part of the existing 03:00 UTC daily cleanup function. (Round 2 / Task 3)
46. `rate_limit_attempts` table added for server-side rate-limiting of `pair-device` (per-IP) and `recover-device` (per-Android-ID). 24-hour retention via the daily cleanup. Replaces the previous "(or Edge Function KV store)" hand-waving with a concrete substrate. Service-role only. (Round 2 / Task 3)
47. FCM payload schema locked to data-only `{ type, operator_id, trigger }`. Push never carries content; it is a sync trigger only. Dispatch filter is `activation_state = 'active'` AND `fcm_token IS NOT NULL`. (Round 2 / Task 3)
48. Hard-delete language for routes removed from the planning suite. Routes are not hard-deleted on a fixed schedule in the initial release. The previous `journey_summaries.route_id` and `devices.active_route_id` justifications referring to a 90-day retention window are removed; the `ON DELETE SET NULL` constraint on `devices.active_route_id` is retained as defence in depth. Updates decision #28. (Round 2 / Task 3)
49. Custom access token hook registration documented as a manual Supabase Auth console action, NOT a database migration. Without the hook, every issued JWT lacks the `operator_id` custom claim, and every operator-scoped RLS policy silently returns empty (for both dashboard and tablet sessions). Added as step 4 in the Data-Architecture §12 setup checklist; called out in both CLAUDE.md Setup Notes and Known Gotchas. (Round 2 / Task 3)
50. Anonymous Supabase Auth user accumulation acknowledged as a deferred operational concern. Each successful tablet pairing creates an anonymous Supabase Auth user; there is no automatic cleanup in the initial release. Deregistering or replacing a tablet leaves the auth user behind. Flagged for future operational hardening; not blocking for initial release. (Round 2 / Task 3)
51. CLAUDE.md propagation rule added to WORKFLOW.md §13.7: any re-planning event that bumps the version of any frozen document must include, as the final task of that event, a sweep of the relevant CLAUDE.md file(s); the frozen-documents rule does not re-engage until both CLAUDE.md files carry the same version as the frozen suite. Lesson learned from round-2 adversarial finding 1 (round-1's seven-task campaign closed without a CLAUDE.md sweep, leaving the builder-facing summaries stale relative to v3.7). (Round 2 / Task 4)

---

## 3. Session History

In chronological order. Append only.

### Planning Sessions (April–May 2026)

The complete planning phase was conducted in one long architect chat across multiple sessions. Outputs:

- Revised PRD (v3.0) with the dashboard surface added, all v1/v2 hedging removed, tablet-side route creation removed, and the stylised tube-map view added
- Revised Data-Architecture (v3.0) reflecting Supabase Auth for both surfaces, pairing-code flow, the `naptan_stations` table with full-text search, and three new Edge Functions
- Revised Compliance Mapping Matrix (v3.0) updated to the new FR-AT-* / FR-WD-* numbering
- New WORKFLOW.md (v1.0) codifying the three-window setup, plan-mode-first ritual, effort levels, commit cadence, the CONTEXT.md update ritual, and the frozen-documents rule
- Two CLAUDE.md files, one per repo

Next session: starts in a new architect chat. First task is initialising the dashboard repository and beginning Stage 1.

### Post-Adversarial-Review Re-Planning Campaign (May 2026)

After the initial five planning documents were drafted, an adversarial review identified structural weaknesses across seven areas: stop-detection model, online/offline and sync story, device identity and auth, `is_approved` boolean overloading, compliance gaps (driver panel occlusion, multi-tablet audio, missing visual alert, calibration verification gap), features that look small but aren't (Kiosk Level 2, NaPTAN tablet bundle, upload-sync scaffolding), and frozen-document contradictions (WORKFLOW vs CONTEXT on repo count, no Room migration policy). A seven-task re-planning campaign addressed each area in sequence: Task 1 (stop detection), Task 2 (hail-and-ride and diversion model), Task 3 (pre-rendered audio architecture), Task 4 (auth, device identity, status enum), Task 5 (FCM, heartbeat, online status), Task 6 (scope cuts), and Task 7 (compliance gaps, WORKFLOW fixes, smaller schema items, parked contradictions, campaign close-out). All five planning documents reached v3.7. The CLAUDE.md files were not touched during this campaign — a gap caught by the round-2 adversarial review (see below).

### Post-Adversarial-Review Re-Planning Round 2 (May 2026)

A second adversarial review was run on the round-1 v3.7 outputs. It identified ten finding clusters, the most consequential of which was that the two CLAUDE.md files had never been updated during the round-1 campaign and would, on day one of the build, instruct the builder to scaffold features that the v3.7 frozen docs had explicitly cut (bundled tablet NaPTAN, upload-sync, Kiosk Level 2, boolean `is_approved`, on-device TTS, the deprecated `render-route-audio` Edge Function). Other findings included the audio pipeline's lack of cost/latency sanity-checking (no TTS provider locked, no concurrent-edit semantics, no failure surface), the heartbeat ownership being entangled with the GPS service, the FCM-render race, the FR-WD-12 divergence detector firing on every save, the audio-enabled designation being framed as a system-enforced constraint when it can only be honour-system, the lack of crash telemetry, and several smaller items (GMS detection assumption, transient-vs-terminal recovery classification, `status`/`activation_state` name collision, anonymous Auth user accumulation, FCM payload schema unspecified, rate-limit substrate hand-waved).

A four-task round-2 campaign addressed the findings in sequence: **Task 1** (audio pipeline overhaul — `pg_boss` job queue, Google Cloud TTS lock-in to `en-GB-Neural2-B`, version-keyed Storage paths, render-then-FCM ordering, content-hash differential re-render, render-status surface); **Task 2** (bug fixes and operational hardening — heartbeat lifecycle redesign, honest `audio_enabled` framing, mid-journey-suspension grace, Sentry crash telemetry across all three surfaces, journey-summary upload); **Task 3** (missing-detail fixes and small bugs — `audio_enabled` count fix, `rate_limit_attempts` table, FCM data-only payload schema, structural FR-WD-12 divergence via `stops_content_hash`, `journey_state` staleness recovery, audio-download skip on display-only tablets, hard-delete language removed, GMS detection at first run, transient vs terminal recovery classification, `devices.status` → `devices.activation_state` rename, billing/deregistration reconciliation); **Task 4** (CLAUDE.md full sweep + WORKFLOW §13.7 propagation rule + §14 round-2 lessons + CONTEXT.md close-out).

Round 1 produced v3.7; round 2 produced v3.8. All five planning documents and both CLAUDE.md files are now at v3.8. The frozen-documents rule re-engages; the new WORKFLOW §13.7 CLAUDE.md-propagation rule means future re-planning rounds cannot close without a CLAUDE.md sweep. Next: Stage 1 — web dashboard MVP.

---

## 4. Reference Notes

Cross-cutting facts that don't fit cleanly in the sections above but matter for future tasks.

- **Two CLAUDE.md files exist, both at v3.8.** The Android repo's CLAUDE.md covers Kotlin/Hilt/Room/GPS conventions; the dashboard repo's CLAUDE.md covers Next.js/TypeScript/Supabase conventions. Both were brought into the versioning system at v3.8 during the round-2 close-out and now carry the same version number as the four frozen planning documents. References in PRD compliance sections (alert chime FR-AT-27, visual alert FR-AT-65, audio rules FR-AT-28) refer to the Android CLAUDE.md. The CLAUDE.md files are the authoritative builder-facing summaries of architectural rules; their substantive `[Tag] Rule` lists are now the canonical reference for what the builder must and must not do.
- **CONTEXT.md is the most up-to-date document in the project.** When in doubt about current state — what's been decided, what's parked, what's deferred — start here. The frozen planning suite captures the contract at the v3.8 freeze point; CONTEXT.md captures everything that has evolved since.
- **One Supabase project, shared schema.** The dashboard and the Android app use the same Supabase project, the same tables, the same RLS policies. Any schema change affects both surfaces. The architect coordinates schema changes deliberately.
- **NaPTAN is dashboard-only.** The `naptan_stations` table lives in Supabase and is queried only by the dashboard route builder. Tablets hold no NaPTAN data; stop names, coordinates, and NaPTAN IDs travel with routes via `route_stops` (copied at route-creation time on the dashboard).
- **No route hard-deletes in the initial release.** Round 2 removed the 90-day hard-delete language. Routes are soft-deleted (`is_deleted = true`) only. The `ON DELETE SET NULL` on `devices.active_route_id` is retained as defence in depth (decision #28).
- **Custom access token hook is a manual Supabase Auth console step.** Not a migration. Without it, RLS-scoped queries silently return empty. After any project reset, re-register before debugging anything else. (Decision #49 and §12 of Data-Architecture.)
- **Audio pipeline is `pg_boss`-based.** Dashboard → `enqueue-render-job` → `pg_boss` queue → `audio-render-worker` (consumes `render-route-audio` job) → Google Cloud TTS (`en-GB-Neural2-B`, locked) → version-keyed Storage paths → FCM data-only push. Render-then-FCM ordering is mandatory. The synchronous `render-route-audio` Edge Function from round 1 is gone. (Decisions #31, #32, #47.)
- **The frozen documents are read-only for the build.** All five documents and both CLAUDE.md files are at v3.8. If during the build the architect believes a frozen document is wrong, that triggers re-planning, not editing. See WORKFLOW.md §13. WORKFLOW.md §13.7 (new in v3.8) requires any future re-planning round to close with a CLAUDE.md sweep before frozen status re-engages.
- **Round-2 lessons codified in WORKFLOW §14.** Three operational practices to preserve: report-don't-fix contradictions inside a task; run adversarial review on planning artefacts before unfreezing for build; decide review cadence round-by-round rather than committing upfront.
- **Architect chats can stay long.** Unlike claude-code chats, this Anthropic project's chats hold long planning sessions without significant degradation. New architect chats start when the current one feels muddled or hits compression.
- **The pilot customer is a personal contact's bus company.** This is helpful for real-world testing but should not be the sole basis for declaring product-market fit. Plan to have at least one paying customer outside the immediate network before generalising.
