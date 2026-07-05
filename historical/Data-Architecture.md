> **HISTORICAL RECORD — NOT CURRENT AUTHORITY**
>
> This is the original **Data Architecture** for the Passenger Display System — the Supabase schema and RLS model, Edge Functions, Room (on-device) schema, sync sequence, and audio-render pipeline the product was designed and built against. It is frozen at v3.9 (May 2026).
>
> It is retained as the historical architecture record — **not** the authority for present-day design decisions. Current project state and remaining work live in `pds-planning/living/STATE.md`; the narrative history up to the July 2026 workflow migration is in `pds-planning/historical/PROJECT-HISTORY.md`. Several parts of the built system have since diverged from this document (some divergences are recorded in code comments and in PROJECT-HISTORY.md); where this document and the current implementation differ, that divergence is expected and the current implementation is authoritative.

# Data Architecture Document
# Passenger Display System (PDS)

**Version:** 3.9
**Last Updated:** May 2026
**PRD Alignment:** PRD v3.9

## Changelog

### v3.9 (May 2026 — round 3, item 3 of 4)
Round-3 post-adversarial-review re-planning pass, item 3 of 4 (cross-document contradictions, missing details, and operational acknowledgements). Resolves `adversarial-review.md` findings 2, 3, 4, 6, 7, 8, 9, 10, 13, 14, 16, 17, 18, 19 — the non-audio-pipeline cluster. Two CLAUDE.md contradictions (findings 2 and 3) are reported here but the CLAUDE.md side is deferred to item 4 of 4.
- §2.2: Added `recover_failure_count INT NOT NULL DEFAULT 0` and `last_recover_failure_at TIMESTAMPTZ NULLABLE` columns to `devices`. Incremented inside the `recover-device` terminal-failure path; reset to 0 on a successful recover. Consumed by the §10 compound auto-deregistration job.
- §3.1: Added explicit transaction-ordering guarantee — the AFTER INSERT trigger on `auth.users` runs in the same Postgres transaction as the user-row insert, so the `operators` row is committed before Supabase Auth can issue the access token. The `custom_access_token_hook` therefore always sees a committed `operators` row when it fires. No race possible. Resolves round-3 finding 7 (signup half).
- §4.2: Added explicit transaction-ordering guarantee after step 10 — steps 7–9 (devices INSERT, app_metadata update, pairing-code mark-used) commit before `auth.admin.createSession` is called. The `custom_access_token_hook` always sees a committed `devices` row when it fires for the new session. Orphaned devices rows from mid-flow failures are harmless and reaped by §10. Resolves round-3 finding 7 (pair-device half).
- §5.5: Added `diversion_invoked_at_any_point BOOLEAN NOT NULL DEFAULT false` to `journey_state`. One-way latch within a journey — set true on FR-AT-25 diversion-start; never reset mid-journey; reset to default only on a new journey. Resolves round-3 finding 6.
- §5.9 (new): `device_state` single-row Room table for device-level state that the tablet maintains locally between syncs (`audio_enabled`, `last_synced_at`). Replaces the previously-dangling references to "the most recently synced `devices` row" in §7.2. Forward-compatible for future device-level cached state. Resolves round-3 finding 4.
- §7.2: Steps 7 and 8 now read and write `device_state.audio_enabled` explicitly. Step 8 ends with an update of `device_state` (audio_enabled + last_synced_at). Resolves round-3 finding 4 (consumer half).
- §7.8: `diversion_invoked = journey_state.diversion_invoked_at_any_point` evaluated at journey end — covers all four cases (no diversion, active at end, started-and-ended mid-journey, replayed). Implementation-choice paragraph removed; the column is now spec'd. Resolves round-3 finding 6 (consumer half).
- §10: Auto-deregistration is now a **compound condition** — `last_seen_at` older than 60 days AND `recover_failure_count >= 5` with `last_recover_failure_at` in the trailing 60-day window. Distinguishes seasonal/in-repair silence (never accumulates failures, retained indefinitely, billing-excluded after 30 days) from operator-decommissioned (actively rejected, auto-deregistered after the threshold). Resolves round-3 finding 13.
- §11.2: Added Sentry quota budget analysis (~140/month expected steady-state at 20-tablet pilot scale; 5,000/month ceiling gives ≈35× headroom). Counts: errors and crashes only; transactions disabled. Quota-warning response specified. Resolves round-3 finding 18.
- §11.2: Strengthened `device_id` breadcrumb language — "UUID identifier with no personal information embedded; permitted as a diagnostic correlation key." Removes ambiguity flagged against the (deferred) Android CLAUDE.md Rule 17 contradiction. Resolves round-3 finding 3 (Data-Architecture half).
- §12: Added a **staging environment** preamble above the checklist — two Supabase projects (production + staging), two Vercel targets, six Sentry DSNs (per-surface-per-environment). Resolves round-3 finding 17.
- §12: Added an **Order dependencies** paragraph after the checklist — custom access token hook before first signup; FCM credentials before first audio-render-worker deploy; Sentry DSN before any build that initialises Sentry. Resolves round-3 finding 19.

### v3.9 (May 2026 — round 3, item 2 of 4)
Round-3 post-adversarial-review re-planning pass, item 2 of 4 (audio pipeline tightening). The two empirically-verified findings from the round-3 spike (`spike-records/round-3/`) — pg_boss-in-Edge-Functions and the Reg 13(4) frequency verification — are encoded as inviolable architecture rules and as a settled compliance position respectively. Five smaller audio-pipeline gaps from `adversarial-review.md` (findings 5, 11, 12, 20) and one spike-derived correction (LINEAR16 vs MP3) are addressed.
- §2.7/§2.8: Audio format switched end-to-end from MP3 (22.05 kHz, ~32 kbps) to LINEAR16 PCM (24 kHz, mono, WAV container). Reason: MP3's psychoacoustic compression discards sub-300 Hz fundamental and parts of the formant zone; LINEAR16 preserves the Reg 13(4)-verified spectrum (Compliance Matrix §13(4)). Storage path file extensions `.mp3` → `.wav`. Storage size estimates revised: ≈ 240 KB per 5 s stop announcement (vs ~25 KB MP3); 10-stop route per version ≈ 2.6 MB; steady-state per route with 3-version retention ≈ 8 MB.
- §4.4: The dashboard's "Generate return route" Server Action explicitly enqueues an audio render job for the new return route via `enqueue-render-job` (parity with route-save), addressing round-3 finding 11.
- §4.6: pg_boss runtime configuration `{ supervise: false, schedule: false }` promoted to an **inviolable architectural rule** with empirical basis in `spike-records/round-3/findings-pg_boss.md`. A new scheduled Edge Function `pgboss-maintain` (`cron: '0 * * * *'`) runs `boss.maintain()` hourly to replace the supervision loop's archival/expiry duties. TTS API call updated to `audioEncoding: 'LINEAR16'`, `sampleRateHertz: 24000`. Target Storage path file extension `.wav`. `enqueue-render-job` gains a server-side deduplication step (FR-WD-21): if a `created` or `active` pg_boss job already exists for the same `(route_id, route_version)`, the function returns the existing job ID without enqueueing a duplicate (suppresses the redundant FCM dispatch named in round-3 finding 20).
- §4.7: Storage cleanup retains **three** most-recent versions per route, up from two (round-3 finding 12). Rationale: covers rapid "edit, notice mistake, edit again" iteration within a single cleanup cycle. Residual assumption (more than 3 versions iterated within a 24-hour cycle produces transient `AUDIO_FILE_MISSING` on tablets holding an intermediate version, self-heals on next sync) named explicitly.
- §5.1, §6.3, §6.4, §7.x, §8.3, §8.4: Audio file extension `.mp3` → `.wav` throughout (Room hash-column descriptions, bundled APK asset table, local file storage layout, sync download prose, announcement flow diagrams, data-flow diagram).
- §6.3 bundled-asset table: file sizes revised for LINEAR16/WAV (≈ 10× larger than MP3). Note added that bundled re-rendering is a Stage 2 build task using the same `en-GB-Neural2-B` / LINEAR16 / 24 kHz configuration as `audio-render-worker`; this section specifies the spec, not the build.
- §10 retention table: "Two most-recent versions" → "Three most-recent versions" (cross-references §4.7).
- §11.2: LINEAR16 / 24 kHz / WAV output note added under Google Cloud TTS dependency; cites Reg 13(4) preservation rationale.
- §12 setup checklist: `pgboss-maintain` (`0 * * * *`) added to step 5 alongside `audio-render-worker` and `audio-cleanup-worker`. Step 6 (Reg 13(4) verification) updated to record the verification as already complete (see `spike-records/round-3/findings-tts-frequency.md` and Compliance Matrix §13(4)); no pre-deployment voice verification work remains.

### v3.8 (May 2026 — item 3 of 4)
Round-2 post-adversarial-review re-planning pass, item 3 of 4 (missing-detail fixes and small bugs).
- §2.2: Renamed `devices.status` → `devices.activation_state` with an explicit CHECK constraint, disambiguating from `operators.status`. Updated `active_route_id` description to drop the 90-day hard-delete reference (no scheduled hard-deletes in the initial release — see §10).
- §2.4: Added `stops_content_hash TEXT NULLABLE` column to `routes` — SHA-256 of the canonical stop-list serialisation. Drives FR-WD-12 structural divergence detection (replaces the old `updated_at > last_synced_with_return` comparison, which fired on every save). `last_synced_with_return` is retained as a soft audit signal but is no longer the divergence trigger.
- §2.6: Dropped 90-day hard-delete reference from `journey_summaries.route_id` description.
- §2.8: Storage "Deleted" lifecycle bullet rephrased — hard-deletion is a manual administrative action, not part of automated retention.
- §2.9 (new): `rate_limit_attempts` table — concrete schema for the rate-limit storage substrate previously referenced as "(or Edge Function KV store)" in §4.2 and §4.3. Service-role only; 24-hour retention via the existing 03:00 UTC daily cleanup.
- §3.2: Added "Anonymous auth-user accumulation" acknowledgement — every pair-device creates a new `auth.users` row that survives deactivation. MVP-acceptable, future cleanup task acknowledged out-of-scope.
- §4.2 `pair-device`: Rewrote step 6/6a around the `audio_enabled` count bug. Count of existing devices is now queried *before* the INSERT, then `audio_enabled = (existing_count = 0)` is set directly in the INSERT — no post-insert UPDATE. Added an integration-test requirement that pairs the first device and asserts `audio_enabled = true`. Rate-limit subsection updated to reference §2.9 explicitly (dropped the KV-store hedge).
- §4.3 `recover-device`: `devices.status = 'inactive'` → `devices.activation_state = 'inactive'`. Added a "Failure classification" subsection consumed by PRD FR-AT-04 (transient vs terminal). Rate-limit subsection updated to reference §2.9 explicitly.
- §4.4 `replace_route_with_stops`: Added step 4 — compute `stops_content_hash` from the just-inserted stops and write it to `routes`. Specifies the canonical hash form and why `updated_at` and `stops_content_hash` diverge for non-structural edits.
- §4.7 Storage cleanup: Added a soft-deleted-routes paragraph and rephrased the hard-delete cross-reference as a manual administrative action rather than a scheduled retention pass.
- §5.4 `journey_events`: Added `JOURNEY_AUTO_CLEARED`, `DIVERSION_REPLAYED`, `GMS_OVERRIDE_ACKNOWLEDGED` event types with their respective `detail` values.
- §5.5 `journey_state`: Added recovery staleness rules and the constants `JOURNEY_STATE_MAX_AGE_HOURS = 8` and `JOURNEY_EVENT_RECENCY_THRESHOLD_HOURS = 1`. Stale state is auto-cleared on launch rather than resumed; diversion replay on non-empty `journey_skipped_stops` is folded into the resume path.
- §7.2 sync algorithm: Added the `audio_enabled` guard at the top of the audio-download step — display-only tablets (`audio_enabled = false`) skip audio download entirely. Step 8 also detects `audio_enabled` flips `false → true` and resets the sync cursor so the next sync back-fills audio.
- §7.4 full sync sequence: Cross-references the §7.2 `audio_enabled` guard.
- §7.6 FCM: Concrete payload schema specified — data-only Android messages with `type`, `operator_id`, `trigger` keys. Dispatch filter updated to `activation_state = 'active'` AND `fcm_token IS NOT NULL`.
- §10 retention table: Removed the "could be hard-deleted after 90 days" language. Replaced "Inactive devices" row with "Active vs auto-deregistered devices" pinned to **30-day heartbeat-billable / 60-day auto-deregister**, reconciling PRD §1.4 (billing window) and the previously-uncoordinated deregistration policy. Added a row acknowledging anonymous-auth-user accumulation. Added a staleness-recovery cross-reference to the journey-state row.
- §12 setup checklist: Added a new step 4 for registering the custom access token hook in Supabase Auth — a manual console action that does not travel in migrations and whose absence causes silent RLS failures. Added `rate_limit_attempts` cleanup and 60-day auto-deregistration to the existing 03:00 UTC scheduled function.

### v3.8 (May 2026 — item 2 of 4)
Round-2 post-adversarial-review re-planning pass, item 2 of 4 (bug fixes and operational hardening: heartbeat lifecycle redesign, mid-journey suspension grace, journey-summary upload, Sentry telemetry).
- §2.6 (new): `journey_summaries` Supabase table — per-journey count metrics (no PII, no location traces). RLS scoped by `operator_id`.
- §2.7 → §2.8 renumbered: previous §2.6 `naptan_stations` becomes §2.7; previous §2.7 Storage becomes §2.8.
- §5.8 (new): `journey_summaries_pending` Room table for queued journey summaries awaiting upload. The `needs_upload` flag here is the **only** legitimate use of upload-sync metadata in the initial release; this is a deliberate, scoped exception to the v3.6 removal of the broader upload-sync scaffolding (which targeted `routes` and per-stop metadata, not journey-summary entities).
- §7.4 Sync Sequence: Operator-status check no longer locks the UI when `journey_state.is_active = true`. The captured status is recorded in `sync_metadata` and acted on at journey end. Mid-journey suspension is honoured at journey end, not mid-journey.
- §7.7 Heartbeat Mechanism: Rewritten. Heartbeat ownership moves from the foreground GPS service to an application-level lifecycle observer (Hilt-injected singleton observing `ProcessLifecycleOwner`). Closes the "foregrounded but no active journey" third-state gap. The foreground GPS service is no longer the heartbeat owner — it exists for stop detection during journeys only.
- §7.8 (new): Journey-summary upload at journey end — before clearing `journey_state`, the tablet writes a row to `journey_summaries_pending`; a small upload step (during the next sync) inserts into Supabase `journey_summaries` and clears `needs_upload`.
- §8.5: Heartbeat flow diagram updated for the new lifecycle-observer ownership.
- §9: Entity Relationship Summary updated — `journey_summaries` added to Supabase tables list; `journey_summaries_pending` added to Room tables list.
- §10: Data retention table — added `journey_summaries` (indefinite in Supabase, operator-visible) and `journey_summaries_pending` (cleared on successful upload).
- §11.2 (new): Third-Party Service Dependencies subsection — Sentry (sentry.io) added as the crash-reporting / error-tracking SDK across Android, dashboard, and Edge Functions. DSN per project, one-off operational setup at Stage 2/3 start.
- §12: Added `SENTRY_DSN_ANDROID`, `SENTRY_DSN_DASHBOARD`, `SENTRY_DSN_EDGE` environment variables (three separate DSNs, one per project).

### v3.8 (May 2026 — item 1 of 4)
Round-2 post-adversarial-review re-planning pass, item 1 of 4 (audio pipeline overhaul: pg_boss job queue, Google TTS lock-in, version-keyed Storage paths, render-then-FCM, content-hash differential re-render, render-status surface).
- §2.4: Added `audio_render_status` (`pending`/`ok`/`failed`), `audio_render_error`, and `audio_announcement_hash` columns to the routes table.
- §2.5: Added `audio_content_hash` column to route_stops for differential re-rendering.
- §2.7: Storage path scheme is now version-keyed (`{operator_id}/{route_id}/{route_version}/...`) where `route_version = routes.updated_at` epoch millis. Each save produces fresh paths; cleanup retains two most recent versions.
- §4.4: Saving a route now enqueues a pg_boss job via `enqueue-render-job`; the dashboard does not fire FCM on save.
- §4.6: Replaced the synchronous `render-route-audio` Edge Function with a pg_boss-backed job queue and scheduled worker (`audio-render-worker`). Locked TTS voice to Google Cloud TTS `en-GB-Neural2-B`. Specified retry policy, content-hash differential rendering, version-path copy for unchanged stops, partial-failure resumption, and render-then-FCM dispatch.
- §4.7 (new): Daily Storage cleanup job (`audio-cleanup-worker`) keeps the two most recent versions per route.
- §7.2, §7.4: Tablet audio download uses version-keyed paths derived from `routes.updated_at`. Tablet skips audio download for routes whose `audio_render_status = 'failed'`.
- §7.6: FCM dispatch is now triggered by the render worker on success, not by a routes-table trigger.
- §8.4: Data-flow diagram replaced — render runs to completion before FCM fires; rendering is asynchronous and observable via `audio_render_status`.
- §10: Audio retention clarified — two most recent versions per route.
- §11: Implementation notes updated for content-hash differential rendering, locked voice, and pg_boss schema ownership.
- §12 (new): Centralised environment variables, including `GOOGLE_TTS_API_KEY`.

### v3.7 (May 2026)
Post-adversarial-review re-planning pass, item 7 of 7 (compliance, WORKFLOW, smaller items, and campaign close-out).
- §2.2: Added `audio_enabled BOOLEAN NOT NULL DEFAULT true` to devices table for multi-tablet audio designation. Added `ON DELETE SET NULL` to `active_route_id` FK to survive route hard-deletion.
- §2.3: Changed `device_pairing_codes` primary key from `code TEXT` to `id UUID DEFAULT gen_random_uuid()`; demoted `code` to a regular UNIQUE indexed column. Decouples row identity from the code value.
- §2.4: Added `last_synced_with_return TIMESTAMPTZ NULLABLE` to routes table for return-route divergence detection (FR-WD-12).
- §4.1: Updated `generate-pairing-code` to insert a UUID PK row.
- §4.2: Updated `pair-device` to look up pairing code by `code` column (not PK); added step 6a to set `audio_enabled` based on existing device count.
- §5.5: Added note that single-row `journey_state` is correct for the initial release; shared-state future requires redesign and is deliberately deferred.
- §7.7: Added OEM best-effort caveat for WorkManager idle-path heartbeat on hostile-OEM tablets.

### v3.6 (May 2026)
Post-adversarial-review re-planning pass, item 6 of 7 (scope cuts: Kiosk Level 2 deferred, tablet NaPTAN bundle removed, upload-sync scaffolding removed).
- §5.3: Removed Room `naptan_stations` table. Tablet does not bundle or store NaPTAN data; stop names and coordinates travel with routes via `route_stops` (copied at dashboard route-creation time).
- §6.3: Removed `naptan_stations.json.gz` bundled asset row.
- §7.3: Removed "Sync Algorithm — Upload" section. Sync is download-only in the initial release.
- §7.4: Removed upload step from full sync sequence. Removed "abort download on upload failure" language. Renumbered steps.
- §7.5: Removed speculative future-upload language from Conflict Resolution.
- §5.1: Removed `needs_upload` placeholder note.
- §4.4: Removed placeholder note about tablet route uploads.
- §9: Removed `naptan_stations` from Local (Room) tables list.
- §11: Removed "Upload-before-download" implementation note.
- §2.7: Removed NaPTAN asset size comparison (asset no longer exists on tablet).

### v3.5 (May 2026)
Post-adversarial-review re-planning pass, item 5 of 7 (sync triggers, FCM, and fleet status).
- §2.2: Added `fcm_token TEXT NULLABLE` column to devices table (no longer elided). Updated `last_seen_at` description to reference heartbeat.
- §7.1: FCM re-framed as a core sync trigger (not an optimisation); all three triggers now described with distinct roles.
- §7.6: Removed "This is an optimisation" framing. Updated token registration to reference `devices.fcm_token` in §2.2.
- §7.7 (new): Heartbeat Mechanism — 2-minute lightweight `last_seen_at` update, independent of route sync, two-path implementation.
- §8.5 (new): Heartbeat Flow diagram.

### v3.4 (May 2026)
Post-adversarial-review re-planning pass, item 4 of 7 (auth, device identity, and security).
- §2.1: Replaced `is_approved BOOLEAN` with `status TEXT` enum (`pending`/`active`/`suspended`). Added `admin_notified_at` column for email retry tracking.
- §2.2: Added `device_secret_hash TEXT NOT NULL` column to devices table. Updated `android_id` description.
- §3.1: Added email retry behaviour (one retry after 10 minutes via scheduled Edge Function). Added pending-operators SQL query pattern.
- §3.2: `pair-device` now generates a 256-bit device secret; plaintext returned once to tablet; SHA-256 hash stored in `devices.device_secret_hash`. `pair-device` stamps `app_metadata` for custom claims hook.
- §3.3: `recover-device` now requires both `android_id` and `device_secret`; server hashes and compares.
- §3.4a (new): JWT custom claims mechanism — Supabase Auth custom access token hook stamps `operator_id` and `device_id` into every JWT.
- §3.4: Rewrote RLS policies to use JWT claims directly instead of subquery joins into `devices` or `operators`.
- §4.1: Updated `is_approved` check to `status = 'active'`.
- §4.2: Added device secret generation. Added rate limiting specification.
- §4.3: Added device secret verification. Added rate limiting specification. Updated input signature.
- §4.5: Updated operator resolution to use JWT claim.
- §6.1: Added `device_secret` to EncryptedSharedPreferences.
- §6.2: Updated `android_id` description.
- §7.4: Updated sync status check to discriminate `pending` vs `suspended`.
- §8.1: Updated Operator Signup Flow diagram references.

### v3.3 (May 2026)
Post-adversarial-review re-planning pass, item 3 of 7 (audio architecture — pre-rendered files).
- §2.7: New section — Supabase Storage bucket `route-audio` for per-route audio files. Path scheme, access model, and lifecycle documented.
- §4.4: Added note that saving a route triggers async audio rendering via `render-route-audio`.
- §4.6: New Edge Function — `render-route-audio`. Renders route announcement and per-stop "Next stop" files server-side; stores in Supabase Storage.
- §5.4: Replaced `TTS_FAILURE` event type with `AUDIO_FILE_MISSING` and `AUDIO_PLAYBACK_ERROR`.
- §6.3: Added five bundled audio announcement files (termination, H&R start/end, diversion start/end). Updated asset list.
- §6.4: New section — local audio file storage on the tablet (`filesDir/audio/`). Covers path scheme, cleanup, and journey-start gating.
- §7.2: Added step 6a — audio file download after route sync.
- §7.4: Updated full sync sequence to include audio file download step.
- §8.3: Removed TTS references from Active Journey Data Flow; replaced with pre-rendered audio file playback.
- §8.4: Extended Route Sync Flow diagram to show `render-route-audio` and tablet audio download.
- §11: Removed TTS implementation notes; added audio file presence gating and outdating notes.

### v3.2 (May 2026)
Post-adversarial-review re-planning pass, item 2 of 7 (hail-and-ride and diversion data model).
- §2.5: Added `segment_type` column to Supabase `route_stops`. Values: 'scheduled' (default) or 'hail_and_ride'.
- §4.4: Extended `p_stops` payload clarification to include `segment_type`.
- §5.2: Added matching `segment_type` column to Room `route_stops` entity.
- §5.4: Added `HAIL_AND_RIDE_SECTION_STARTED`, `HAIL_AND_RIDE_SECTION_ENDED`, `DIVERSION_STARTED`, `DIVERSION_ENDED`, `STOP_SKIPPED` event types.
- §5.5: Added note about `journey_skipped_stops` adjunct table.
- §5.7: New `journey_skipped_stops` local-only table for transient diversion-skip state.
- §8.3: Extended Active Journey Data Flow to reflect H&R section transitions and diversion skip logic.

### v3.1 (May 2026)
Post-adversarial-review re-planning pass, item 1 of 7 (stop detection).
- §2.5: Added `proximity_radius_meters` column to Supabase `route_stops`. Default 200m. Set per stop on the dashboard.
- §4.4: Clarified that `p_stops` JSON array objects include `proximity_radius_meters`.
- §5.2: Added matching `proximity_radius_meters` column to Room `route_stops` entity.
- §5.4: Added `STOP_PASSED_WITHOUT_DETECTION` event type and `GPS_INFERRED` trigger method for two-stop look-ahead.
- §6.2: Removed global `proximity_radius_meters` from SharedPreferences. Radius is now per-stop.
- §8.3: Rewrote Active Journey Data Flow to reflect two-stop look-ahead and GPS accuracy gate.

---

## 1. Overview

This document defines every database table, field, relationship, and constraint for both the cloud database (Supabase / PostgreSQL) and the on-device database (Android Room). It also defines the sync strategy, authentication approach, data isolation model, and storage decisions for non-database data.

**Core principles:**

- **Offline-first on the tablet.** The tablet reads exclusively from Room during operation. Supabase is the cloud source of truth for routes, devices, and reference data shared across an operator's fleet. Sync happens in the background when connectivity is available and never blocks any user-facing operation on the tablet.
- **Dashboard reads and writes Supabase directly.** The web dashboard is online by design. It queries and mutates Supabase through the Supabase JS client. There is no offline mode for the dashboard.
- **Uniform auth.** Both dashboard users and tablets authenticate via Supabase Auth. Dashboard users authenticate with email and password. Tablets authenticate as anonymous Supabase Auth users created during pairing. Row-Level Security policies key off `auth.uid()` for both surfaces.
- **All timestamps in UTC.** Room stores epoch millis (LONG). Supabase stores `TIMESTAMPTZ`. Conversion to local UK time (GMT/BST) happens only at the UI layer.
- **Routes and stops are synced as a unit.** Stops are never independently created, modified, or synced. When a route is uploaded or downloaded, its entire stop list is replaced atomically. The architecture is simple and correct for the dashboard-only authoring model.

---

## 2. Supabase (PostgreSQL) Schema

All tables use UUID primary keys generated by Postgres or by the dashboard client. All timestamps are `TIMESTAMPTZ` in UTC. RLS is enabled on every table.

### 2.1 operators

One row per bus company. Each operator is linked to a Supabase Auth user account (the dashboard login).

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PK, DEFAULT gen_random_uuid() | Unique operator identifier |
| user_id | UUID | NOT NULL, FK → auth.users(id), UNIQUE | The Supabase Auth user representing this operator's dashboard login |
| company_name | TEXT | NOT NULL | Company display name (e.g., "Smith's Bus Company") |
| status | TEXT | NOT NULL, DEFAULT 'pending', CHECK (status IN ('pending', 'active', 'suspended')) | Operator account lifecycle state. `'pending'`: signed up but not yet approved by the system administrator. `'active'`: approved and fully functional. `'suspended'`: was previously active but has been suspended (e.g., for non-payment). Dashboard and tablet UI vary by state — see FR-WD-04, FR-WD-05, FR-AT-60. |
| admin_notified_at | TIMESTAMPTZ | NULLABLE, DEFAULT NULL | Timestamp when the signup notification email was successfully sent to the system administrator. NULL if notification has not yet succeeded. Used by the email retry mechanism (§3.1). |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | When the operator signed up |

**Indexes:** `user_id` (for fast `auth.uid()` lookups in RLS policies).

**Signup flow:** When a new user signs up via Supabase Auth, a database trigger creates the corresponding `operators` row with `status = 'pending'` and `company_name` taken from the signup form (stored as Supabase Auth user metadata, then copied here). The same trigger sends an email notification to the system administrator.

### 2.2 devices

One row per registered tablet. Devices are linked to anonymous Supabase Auth users created during pairing.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PK, DEFAULT gen_random_uuid() | Unique device identifier |
| user_id | UUID | NOT NULL, FK → auth.users(id), UNIQUE | The anonymous Supabase Auth user for this device. Created by the `pair-device` Edge Function. |
| operator_id | UUID | NOT NULL, FK → operators(id) | Owning operator |
| display_name | TEXT | NOT NULL, DEFAULT 'New Device' | Operator-assigned human-readable name (e.g., "Bus #42"). Default name appended with sequence number on creation. |
| android_id | TEXT | NOT NULL | Settings.Secure.ANDROID_ID from the tablet. Passed to `recover-device` as the device row lookup key (non-secret identifier). Authentication of the recovery request relies on `device_secret_hash`, not the Android ID alone. |
| device_secret_hash | TEXT | NOT NULL | SHA-256 hash (hex-encoded) of the 256-bit device secret generated at pairing time by `pair-device`. The plaintext secret is returned to the tablet once in the pairing response and stored in EncryptedSharedPreferences. The server never stores or logs the plaintext. Used by `recover-device` to authenticate recovery requests. |
| app_version | TEXT | NULLABLE | App version string at last sync |
| activation_state | TEXT | NOT NULL, DEFAULT 'active', CHECK (activation_state IN ('active', 'inactive')) | Device activation state. `'active'`: device is registered and counts toward billing (subject to the 30-day heartbeat window — see §10). `'inactive'`: set by the dashboard's "Deactivate device" action (PRD FR-WD-17), by `Deregister Device` on the tablet (PRD FR-AT-50), or automatically after 60 consecutive days without a heartbeat (see §10 auto-deregistration rule). Inactive devices are rejected by `recover-device` (§4.3) and do not count toward billing. **Renamed from `status` in v3.8 item 3 of 4** to disambiguate from `operators.status` — having two columns named `status` with different semantics (operators is a three-state lifecycle enum; this is a two-state activation flag) was a code-review hazard. The explicit CHECK constraint is database-enforced. |
| registered_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | When the device first paired |
| last_seen_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Updated on every successful sync and by the heartbeat mechanism (§7.7). Drives online/offline status in the dashboard. A device is considered online if this timestamp is within the last 5 minutes. |
| fcm_token | TEXT | NULLABLE | The FCM registration token for this device. Set during the first successful sync after pairing, when the tablet registers with Firebase and stores the token here. Updated automatically when Firebase rotates the token. Nullable for devices that have not yet completed their first sync. Used by the route-change Edge Function (§7.6) to deliver push notifications. |
| audio_enabled | BOOLEAN | NOT NULL, DEFAULT true | Whether this device produces audio output. `true` for the designated primary (audio) tablet; `false` for display-only tablets. The DEFAULT of `true` is overridden by `pair-device` on every insert — the function counts existing devices for the operator *before* inserting and sets `audio_enabled = (existing_count = 0)` directly in the INSERT (see §4.2 steps 6–7). The first tablet paired to an operator becomes the audio device; every subsequent tablet defaults to `false`. Configurable from the dashboard fleet view. Tablets read this value from their device record after each sync and apply it immediately: if `false`, all audio output (announcements, alert chime, Bluetooth keep-alive) is suppressed for the session, AND audio downloads are skipped entirely (§7.2 step 7 `audio_enabled` guard); visual display continues normally. A flip from `false → true` is detected during sync (§7.2 step 8) and triggers a cursor reset so audio is back-filled on the next sync. |
| active_route_id | UUID | NULLABLE, FK → routes(id) ON DELETE SET NULL | The route currently active on this device. Updated by the tablet on journey start/end. Used by the dashboard's fleet view. Nullable when no journey is in progress. `ON DELETE SET NULL` ensures that if a route row is ever hard-deleted from Supabase (a manual administrative action — the initial release does not perform scheduled hard deletes, see §10 data retention), this FK reference becomes null rather than causing a constraint violation or cascading device deletion. |
| recover_failure_count | INT | NOT NULL, DEFAULT 0 | Count of consecutive **terminal** `recover-device` failures for this device (HTTP 404 device-row-missing, HTTP 401 secret-mismatch, or response indicating `activation_state = 'inactive'` / operator-disabled — see §4.3 failure classification). Incremented inside the `recover-device` terminal path; **reset to 0 on every successful** `recover-device`. Not touched by transient failures (HTTP 5xx, network errors, 429). Consumed by the §10 compound auto-deregistration job to distinguish "tablet is being actively rejected by the backend" (failures accumulate → eligible for auto-deregistration after the 60-day threshold) from "tablet is simply silent" (no failures accumulate → never auto-deregistered, retains `devices` row indefinitely). Diagnostic only; not exposed to the dashboard fleet view. |
| last_recover_failure_at | TIMESTAMPTZ | NULLABLE | Timestamp of the most recent terminal `recover-device` failure. Overwritten on every increment of `recover_failure_count`. Used together with the count by the §10 cleanup job to verify that the failures fall within the trailing 60-day window (a stale failure count from a year ago is not evidence of current rejection). NULL until the first terminal failure ever recorded for this device. |

**Indexes:** `user_id`; `(operator_id, activation_state)` composite; `android_id` (for recovery lookups); `fcm_token` (for push dispatch lookups).

### 2.3 device_pairing_codes

Short-lived single-use codes used to pair a new tablet with an operator account.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PK, DEFAULT gen_random_uuid() | Unique row identifier. Using a UUID PK decouples row identity from the code value, avoiding entanglement between cleanup, audit retention, and the collision-check logic. |
| code | TEXT | NOT NULL, UNIQUE | 6-digit numeric code, generated by the `generate-pairing-code` Edge Function. The `UNIQUE` constraint prevents duplicate live codes. `pair-device` looks up rows by this column, not by PK. |
| operator_id | UUID | NOT NULL, FK → operators(id) | The operator who generated this code |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | When the code was generated |
| expires_at | TIMESTAMPTZ | NOT NULL | When the code becomes invalid (10 minutes after creation) |
| used_at | TIMESTAMPTZ | NULLABLE | Set to `now()` when the code is consumed by `pair-device`. Used codes remain in the table briefly for audit, then are cleaned up. |

**Indexes:** `(code)` (for `pair-device` lookup by code value); `(expires_at)` (for cleanup queries); `(operator_id, created_at)` (for the dashboard's recent-codes view).

**Cleanup:** A scheduled function runs hourly to delete rows where `expires_at < now() - interval '1 hour'`. This is housekeeping; expired or used codes are rejected by `pair-device` regardless.

### 2.4 routes

Route definitions. Each route belongs to one operator. Routes are authored exclusively on the dashboard.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PK | Generated client-side by the dashboard |
| operator_id | UUID | NOT NULL, FK → operators(id) | Owning operator |
| name | TEXT | NOT NULL | Display name (e.g., "Newcastle to Carlisle") |
| route_number | TEXT | NULLABLE | Optional route number |
| direction | TEXT | NULLABLE | Direction label: "Outbound", "Return", or custom |
| return_route_id | UUID | FK → routes(id) DEFERRABLE INITIALLY DEFERRED, NULLABLE | Links to the return route. Deferrable so an outbound + return pair can be inserted in a single transaction. |
| last_synced_with_return | TIMESTAMPTZ | NULLABLE, DEFAULT NULL | Timestamp set on both routes when a return route is generated (FR-WD-12). Retained as a soft audit signal of when the return was last reconciled. **As of v3.8 item 3, this column is no longer the divergence trigger** — divergence detection now compares `stops_content_hash` between the route and its linked return (see column below and PRD FR-WD-12). NULL if no return has ever been generated for this route. |
| stops_content_hash | TEXT | NULLABLE | SHA-256 hex digest of the canonical serialisation of this route's ordered stop list. Canonical form: for each stop in `stop_order` ascending, the string `"{naptan_id}|{stop_order}"`; the strings joined by newline; UTF-8 bytes hashed with SHA-256; lowercase hex output. Used by the return-route divergence check (PRD FR-WD-12) to detect *structural* changes (stops added, removed, or reordered) without firing on cosmetic edits (direction-label tweak, `route_number` change). The hash is computed and stored by the `replace_route_with_stops` RPC (§4.4) on every save. NULL until the first save under v3.8 item 3. Existing routes acquire a hash on their next save. |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Server-assigned via trigger on every INSERT or UPDATE. Used as the sync cursor and as the `route_version` segment of the Storage path scheme (§2.8), expressed as epoch milliseconds. |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT false | Soft delete flag. Deleted routes remain for sync propagation. |
| audio_render_status | TEXT | NOT NULL, DEFAULT 'pending', CHECK (audio_render_status IN ('pending', 'ok', 'failed')) | Current state of the audio render job for this route version. `pending` immediately after `replace_route_with_stops`. Flipped to `ok` by the audio render worker on successful completion (§4.6); flipped to `failed` after the worker exhausts its retries. The dashboard surfaces this status in the route list (PRD FR-WD-13); the tablet uses it to skip audio download for failed routes (§7.4). |
| audio_render_error | TEXT | NULLABLE | Last error message captured when `audio_render_status = 'failed'`. NULL when status is `pending` or `ok`. Cleared back to NULL by the worker on a successful re-render. Surfaced on the dashboard for diagnostic purposes. |
| audio_announcement_hash | TEXT | NULLABLE | SHA-256 hex of the route-announcement text (`"This bus is the [route.name] service to [last_stop.stop_name]."`) that was rendered into the currently-stored `route_announcement.wav`. Compared by the audio render worker against the would-be hash on re-render to skip rendering when unchanged. NULL until the first successful render of this route. See §4.6 for the differential re-render algorithm. |

**Indexes:** `(operator_id, is_deleted, updated_at)` composite for sync queries.

### 2.5 route_stops

Ordered list of stops within a route. Stops are always synced as a complete set with their parent route — they are never independently created, modified, or synced.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PK | Generated client-side by the dashboard |
| route_id | UUID | NOT NULL, FK → routes(id) ON DELETE CASCADE | Parent route |
| naptan_id | TEXT | NULLABLE | NaPTAN identifier. NULL for future custom stops. |
| stop_name | TEXT | NOT NULL | Display name. Copied from naptan_stations at creation so the route survives NaPTAN updates. |
| crs_code | TEXT | NULLABLE | 3-letter station code (e.g., "NCL"). Stored for search result disambiguation. |
| latitude | DOUBLE PRECISION | NOT NULL | WGS84 latitude. Copied from naptan_stations at creation. |
| longitude | DOUBLE PRECISION | NOT NULL | WGS84 longitude. Copied from naptan_stations at creation. |
| stop_order | INTEGER | NOT NULL | 0-based position in the route sequence. Defines announcement order. |
| is_custom | BOOLEAN | NOT NULL, DEFAULT false | False in the initial release. Reserved for the future custom-stop feature. |
| proximity_radius_meters | INTEGER | NOT NULL, DEFAULT 200 | Per-stop GPS proximity radius in metres. The app triggers a stop announcement when a qualifying GPS fix places the bus within this radius. Set per stop in the dashboard route builder; 200m is the pre-filled default. Operators can set a larger value for motorway stops or a smaller value for dense urban stops. |
| segment_type | TEXT | NOT NULL, DEFAULT 'scheduled', CHECK (segment_type IN ('scheduled', 'hail_and_ride')) | The stop's segment classification. 'scheduled' for a normal timetabled stop; 'hail_and_ride' for a stop within a hail-and-ride section. A contiguous run of 'hail_and_ride' stops constitutes a hail-and-ride section. Set per stop in the dashboard route builder; 'scheduled' is the pre-filled default. |
| audio_content_hash | TEXT | NULLABLE | SHA-256 hex of the per-stop announcement text (`"Next stop: [stop.stop_name]."`) that was rendered into the currently-stored `stop_{stop_order}.wav` for this stop. Used by the audio render worker to skip re-rendering when the stop's text has not changed (§4.6 differential re-render). NULL until this stop has been successfully rendered at least once. |

**Indexes:** `route_id` (for loading all stops for a route).

**No updated_at or is_deleted:** Stops do not have their own sync timestamps or soft-delete flags. When a route is synced, its entire stop list is replaced atomically by the `replace_route_with_stops` RPC.

### 2.6 journey_summaries

One row per completed journey. Holds anonymous count metrics derived from the tablet's local `journey_events` log. No PII, no location traces — only counts. Uploaded by the tablet at journey end (see §7.8); the tablet first writes to its local `journey_summaries_pending` table (§5.8) and uploads on next sync.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PK, DEFAULT gen_random_uuid() | Unique summary identifier |
| device_id | UUID | NOT NULL, FK → devices(id) | The device that ran the journey |
| operator_id | UUID | NOT NULL, FK → operators(id) | Owning operator (RLS scope) |
| route_id | UUID | NULLABLE, FK → routes(id) ON DELETE SET NULL | The route that was run. Nullable so summaries survive any future hard-delete of the route — a manual administrative action; the initial release does not perform scheduled hard deletes (see §10 data retention) |
| journey_started_at | TIMESTAMPTZ | NOT NULL | Wall-clock journey start (from local `journey_state.journey_started_at`) |
| journey_ended_at | TIMESTAMPTZ | NOT NULL | Wall-clock journey end |
| stops_announced_count | INTEGER | NOT NULL, DEFAULT 0 | Count of `STOP_ANNOUNCED` events during the journey |
| stops_passed_without_detection_count | INTEGER | NOT NULL, DEFAULT 0 | Count of `STOP_PASSED_WITHOUT_DETECTION` events. A proxy for GPS reliability |
| manual_advances_count | INTEGER | NOT NULL, DEFAULT 0 | Count of `STOP_ANNOUNCED` events with `trigger_method = 'MANUAL'`. A second GPS-reliability proxy |
| gps_lost_events_count | INTEGER | NOT NULL, DEFAULT 0 | Count of `GPS_LOST` events during the journey |
| audio_failures_count | INTEGER | NOT NULL, DEFAULT 0 | Count of `AUDIO_FILE_MISSING` + `AUDIO_PLAYBACK_ERROR` events |
| diversion_invoked | BOOLEAN | NOT NULL, DEFAULT false | True if `journey_skipped_stops` was non-empty at any point during the journey |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Server-side insert timestamp |

**Indexes:** `(operator_id, device_id, journey_started_at DESC)` for the per-device drill-down (PRD FR-WD-23); `(operator_id, journey_started_at DESC)` for any future operator-wide rollups.

**RLS:**
- SELECT: dashboard users where `operator_id = (auth.jwt()->>'operator_id')::uuid`. Device tokens do **not** read this table — devices only write their own rows.
- INSERT: device users where `device_id = (auth.jwt()->>'device_id')::uuid` AND `operator_id = (auth.jwt()->>'operator_id')::uuid`. Constrains a device to inserting only its own summaries under its own operator.
- UPDATE, DELETE: not allowed via policy. The system administrator can act via service role.

**Privacy posture:** This is the only table in the system that summarises per-journey activity. It contains no operator names, no driver names, no passenger information, and no GPS traces. It is deliberately reduced to counts so that the operational benefit (fleet-health drill-down via PRD FR-WD-23) can be obtained without privacy concerns.

### 2.7 naptan_stations

Reference data: all UK NaPTAN entries (railway stations and bus stops). Approximately 400,000 rows. Shared across all operators (not multi-tenant).

| Column | Type | Constraints | Description |
|---|---|---|---|
| naptan_id | TEXT | PK | Official NaPTAN identifier (the ATCO code) |
| stop_name | TEXT | NOT NULL | Stop display name |
| crs_code | TEXT | NULLABLE | 3-letter CRS code (railway stations only) |
| locality | TEXT | NULLABLE | Town or region name for disambiguation |
| latitude | DOUBLE PRECISION | NOT NULL | WGS84 latitude |
| longitude | DOUBLE PRECISION | NOT NULL | WGS84 longitude |
| stop_type | TEXT | NOT NULL | 'RAILWAY' or 'BUS' (derived during import for filtering in the dashboard) |
| search_vector | TSVECTOR | GENERATED ALWAYS AS (to_tsvector('english', stop_name \|\| ' ' \|\| COALESCE(crs_code, '') \|\| ' ' \|\| COALESCE(locality, ''))) STORED | Full-text search index column |

**Indexes:** GIN index on `search_vector`; B-tree on `crs_code`.

**Search:** The dashboard queries via Postgres full-text search using `search_vector @@ websearch_to_tsquery('english', :query)`, ordered by ranking. Returns results within 500ms for typical queries on the full dataset.

**Data loading:** NaPTAN data is imported via SQL editor or `psql` from the Department for Transport's published NaPTAN CSV. The import process is documented in a separate operations runbook; in summary: download the CSV, transform with a one-off script to extract relevant columns and assign `stop_type`, COPY into the table. Re-imports replace existing rows by `naptan_id` UPSERT.

### 2.8 Supabase Storage — Route Audio Files

Pre-rendered audio files for route-specific announcements are stored in Supabase Storage. Fixed
announcement texts that do not vary by route (termination, hail-and-ride start/end, diversion
start/end) are bundled in the APK and are not stored here.

**Bucket:** `route-audio` (private)

**Path scheme (version-keyed):**

| Path | Content |
|---|---|
| `{operator_id}/{route_id}/{route_version}/route_announcement.wav` | "This bus is the [Route Name] service to [Final Stop]." |
| `{operator_id}/{route_id}/{route_version}/stop_{stop_order}.wav` | "Next stop: [Stop Name]." for stop at position N |

`route_version` is `routes.updated_at` expressed as epoch milliseconds (e.g. `1715846400000`).
Each route save produces a freshly-stamped `updated_at` (via the trigger in §3.5) and therefore
a fresh path prefix. Stop order values match `route_stops.stop_order` (0-based). A 10-stop route
produces 11 files per version: one route announcement plus `stop_0.wav` through `stop_9.wav`.

**Race-safety by construction:** Two concurrent saves cannot collide on the same Storage path
because each save yields a different `updated_at`. The newer save enqueues its own render job;
the older job's worker detects staleness on dequeue (§4.6 job-processing step 1) and exits
without rendering.

**Access model:** Storage RLS policies scope read access by operator_id. A device JWT may read
files only under `{their_operator_id}/`. Dashboard users may read files under their own
`{operator_id}/`. Write operations are performed by the `audio-render-worker` Edge Function
(§4.6) running with service-role access.

**Audio format:** LINEAR16 (uncompressed PCM, 16-bit signed little-endian), mono, 24 kHz
sample rate, encapsulated in a canonical PCM WAV container (44-byte header). Chosen end-to-end
to preserve the full frequency spectrum verified for Reg 13(4) (`spike-records/round-3/findings-tts-frequency.md`
and Compliance Mapping Matrix Reg 13(4)). MP3 was previously specified but its psychoacoustic
compression aggressively discards sub-300 Hz energy (loudness contour) and parts of the formant
zone; LINEAR16 has no such loss. The format choice is therefore part of the compliance argument,
not just an implementation detail. Android `MediaPlayer` plays WAV files natively — no
client-side decoder change is required.

**Storage size estimate:** A 5-second stop announcement at LINEAR16 mono 24 kHz is
24000 × 2 × 5 = **240 KB** per file (≈ 10× a comparable MP3). A 10-stop route therefore
produces ≈ 11 × 240 KB ≈ **2.6 MB of audio per version** (vs ~220 KB at the previous MP3
spec). With the **three-version** retention policy (§4.7), the steady-state Storage footprint
per route is approximately 3 × one version ≈ 8 MB. The cost is one-off sync bandwidth on
route changes, not ongoing operation: tablets on cellular WAN with version-keyed sync (each
route fetches audio only when its `route_version` advances) absorb this easily given how
infrequently routes are edited. The trade-off — ~10× the previous Storage footprint per
route — buys a structurally compliant audio path (no lossy compression between the
Reg 13(4)-verified renderer and the tablet speaker).

**Lifecycle:**
- **Created:** When the dashboard saves a route, it enqueues a `render-route-audio` pg_boss job
  (§4.6). The `audio-render-worker` writes files at the new `{route_version}` path prefix.
  The route's `audio_render_status` is `pending` until the worker completes successfully
  (`ok`) or exhausts its retries (`failed`).
- **Updated:** Each route save produces a new `route_version` and therefore a new path prefix.
  The previous version's files remain in Storage until the cleanup job removes them. Tablets
  syncing the updated route pull audio from the new path; tablets that already have the
  previous version's audio downloaded are unaffected until they next sync.
- **Cleaned:** A daily scheduled cleanup job (`audio-cleanup-worker`, §4.7) removes all but
  the three most-recent versions per route.
- **Deleted:** If a route is ever hard-deleted (a manual administrative action; the initial
  release does not perform scheduled hard deletes — see §10 data retention), the entire
  `{operator_id}/{route_id}/` folder — every version — is removed from the bucket.

### 2.9 rate_limit_attempts

Append-only log of rate-limited Edge Function attempts. Used by `pair-device` (§4.2) and `recover-device` (§4.3) to enforce the per-IP and per-Android-ID rate limits specified in those sections. Storing attempt history in a table — rather than in an Edge Function in-memory KV store — keeps the limits durable across Edge Function cold starts and makes the windowed-count queries straightforward SQL.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | UUID | PK, DEFAULT gen_random_uuid() | Row identifier |
| endpoint | TEXT | NOT NULL, CHECK (endpoint IN ('pair-device', 'recover-device')) | Which Edge Function the attempt was against. The CHECK is database-enforced — adding a new rate-limited endpoint in the future requires extending this constraint deliberately. |
| key | TEXT | NOT NULL | Per-attempt identifier. Encoded with a self-describing prefix: `ip:<addr>` for IP-keyed limits (e.g. `ip:203.0.113.42`) or `android:<android_id>` for Android-ID-keyed limits (e.g. `android:abc123def456`). The prefix prevents accidental collisions between the two keying classes and makes ad-hoc diagnostic queries readable. |
| attempted_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | When the attempt happened |
| succeeded | BOOLEAN | NOT NULL | Whether the attempt succeeded. The rate-limit logic counts failed attempts (`succeeded = false`) within the relevant window. The "double-weight for genuinely invalid codes" behaviour described in §4.2 lives in the `pair-device` function code (e.g. by inserting two rows for that class), not in the table schema. |

**Indexes:** `(endpoint, key, attempted_at DESC)` composite — supports the windowed lookup `SELECT count(*) FROM rate_limit_attempts WHERE endpoint = $1 AND key = $2 AND attempted_at > now() - $window AND NOT succeeded` that the rate-limit logic runs on every attempt.

**RLS:** service-role only. Never read or written by clients directly. The Edge Functions hold the service-role key and perform both INSERTs (recording each attempt) and SELECTs (windowed counts) against this table.

**Cleanup:** Rows older than 24 hours are deleted by the existing daily scheduled cleanup at **03:00 UTC** (alongside the `audio-cleanup-worker` schedule — §4.7 — by adding a `DELETE FROM rate_limit_attempts WHERE attempted_at < now() - interval '24 hours'` step to that scheduled function, or by colocating with `device_pairing_codes` hourly cleanup; the project's scheduled-function inventory determines the most convenient host). 24 hours is well beyond the longest rate-limit window in use (`recover-device` per-IP: 1 hour) and keeps the table small even under sustained traffic.

---

## 3. Supabase Authentication and Row-Level Security

### 3.1 Authentication — Dashboard Users

Operators sign up at the dashboard with email, password, and company name. Supabase Auth handles:
- Password hashing (bcrypt)
- Email verification (sent automatically by Supabase)
- Password reset flow
- Session JWT issuance and refresh

A database trigger on `auth.users` fires `AFTER INSERT` to:
1. Create the corresponding `operators` row with `status = 'pending'` and `company_name` from raw user metadata.
2. Call a Supabase Edge Function (or pg_net HTTP request) that sends the signup notification email to the system administrator address. If the send succeeds, the trigger sets `admin_notified_at = now()` on the operators row.

**Ordering guarantee (no race on first session).** The AFTER INSERT trigger runs **inside the same Postgres transaction** as the `auth.users` INSERT performed by Supabase Auth's signup machinery. The `operators` row is therefore committed atomically with the user row. Supabase Auth issues the access token for the new session (which invokes the `custom_access_token_hook`, §3.4a) only **after** that transaction commits; by Postgres MVCC the new `operators` row is invisible to other connections until commit. Consequently, when the hook fires for the very first session, the `operators` row is always present and visible. The previously-flagged race (hook fires before row exists, returning a JWT without an `operator_id` claim) cannot occur with this trigger-in-same-transaction design. No custom signup Edge Function is required to enforce ordering — Postgres transaction semantics give it for free. The signup notification email's send result is an outer side effect that does not affect this guarantee.

**Email retry:** A Supabase scheduled Edge Function (`retry-admin-notification`) runs every 5 minutes. It queries for operators where `status = 'pending'` AND `admin_notified_at IS NULL` AND `created_at < now() - interval '10 minutes'`, attempts to resend the notification, and sets `admin_notified_at = now()` on success. This provides one automatic retry after a transient send failure. Once `admin_notified_at` is set (by either the trigger or the retry), the scheduled function does not send again for that operator. If the retry also fails, no further automatic notification is sent; the pending-operators query below is the fallback.

**Pending operators SQL query:** The database is the authoritative pending queue. The email is the notification trigger, not the source of truth. The system administrator can query all operators awaiting approval at any time:

```sql
SELECT
  o.id,
  o.company_name,
  u.email,
  o.created_at,
  o.admin_notified_at
FROM operators o
JOIN auth.users u ON o.user_id = u.id
WHERE o.status = 'pending'
ORDER BY o.created_at;
```

The dashboard's Supabase client uses the user's session JWT for all queries. RLS policies key off JWT claims (see §3.4a).

### 3.2 Authentication — Tablets

Tablets authenticate as anonymous Supabase Auth users created during pairing. The flow:

1. Fleet manager clicks "Add Device" on the dashboard.
2. Dashboard calls the `generate-pairing-code` Edge Function. The function:
   - Verifies the caller is an authenticated operator with `status = 'active'`
   - Inserts a row into `device_pairing_codes` with a freshly generated 6-digit code, the caller's `operator_id`, and `expires_at = now() + interval '10 minutes'`
   - Returns the code
3. Dashboard displays the code to the user.
4. User enters the code on the tablet's first-run setup screen.
5. Tablet calls the `pair-device` Edge Function with the code and the device's Android ID. The function:
   - Looks up the pairing code; rejects if not found, expired, or already used
   - Rejects if the operator linked to the code has `status != 'active'`
   - Creates a new anonymous Supabase Auth user via the Admin API, setting `app_metadata: { role: 'device', operator_id: '<UUID>' }` (device_id is added after the devices row is created)
   - Generates a cryptographically random 256-bit device secret (`crypto.getRandomValues(new Uint8Array(32))`, base64url-encoded, 43 chars)
   - Computes SHA-256 hash of the plaintext secret (hex-encoded)
   - Inserts a `devices` row linking that anonymous user to the operator from the pairing code, with the Android ID, `device_secret_hash`, and default display name
   - Updates the anonymous user's `app_metadata` to add `device_id` (now that the devices row UUID is known)
   - Marks the pairing code as used
   - Creates a session for the anonymous user via the Admin API (the custom access token hook — §3.4a — fires automatically and stamps `operator_id` and `device_id` claims)
   - Returns `{ access_token, refresh_token, device_id, operator_id, operator_name, device_secret }`. The `device_secret` is the plaintext — returned once only, never stored or logged server-side after this response.
6. Tablet stores the JWT, refresh token, and device secret in EncryptedSharedPreferences. Regular SharedPreferences stores `operator_id`, `operator_name`, `device_id`, and `android_id` (non-sensitive identifiers). From this point, the tablet acts as the anonymous Supabase Auth user for all subsequent queries.

The tablet's Supabase client refreshes the JWT automatically using the refresh token.

**Anonymous auth-user accumulation (known unbounded growth — MVP-acceptable).** Every `pair-device` call creates a new anonymous Supabase Auth user; every device deactivation flips `devices.activation_state` to `inactive` but **leaves the corresponding `auth.users` row in place**. Across the product's lifetime, retired-device auth-user rows accumulate without bound. At MVP fleet size (small numbers of operators, tens of devices each) the growth is irrelevant — Supabase's MAU limits and storage are well clear. At scale this would become a real operational concern: a future cleanup task should periodically delete `auth.users` rows whose corresponding `devices.activation_state = 'inactive'` and whose last activity is older than a threshold (e.g. 12 months). **Out of scope for the initial release;** recorded here so the awareness is not lost.

### 3.3 Refresh Token Recovery

If a tablet sits offline long enough that its refresh token expires (typical Supabase default is 7 days; we extend this in project settings to allow weeks/months), the next sync attempt returns a 401. The tablet detects this and silently calls the `recover-device` Edge Function:

1. The tablet passes its stored Android ID and plaintext device secret. (No JWT — the device is unauthenticated at this point.)
2. The function looks up the `devices` row by `android_id`. Rejects with HTTP 404 if not found.
3. Computes the SHA-256 hash of the provided `device_secret` and compares against the stored `device_secret_hash`. Rejects with HTTP 401 if the hashes do not match. (Uses constant-time comparison to avoid timing attacks.)
4. Rejects if `devices.activation_state = 'inactive'` or if the linked operator has `status != 'active'`.
5. Issues a fresh session for the existing anonymous user (`user_id` on the devices row) via the Admin API. The custom access token hook (§3.4a) fires automatically and stamps `operator_id` and `device_id` claims.
6. Returns `{ access_token, refresh_token, device_id, operator_id }`.

The device row is unchanged; only the auth session is refreshed. The user does not see the pairing screen again.

If `recover-device` fails, the tablet's behaviour depends on whether the failure is **terminal** (device row missing, secret mismatch, device inactive, operator disabled) or **transient** (5xx, network error, rate-limited). Terminal failures wipe cached credentials and return to first-run setup; transient failures retain credentials and retry silently on the next sync. See PRD FR-AT-04 and the "Failure classification" subsection of §4.3 below for the full split.

### 3.4a JWT Custom Claims

Every JWT issued by this system carries `operator_id` (and, for device tokens, `device_id`) as top-level claims. These claims allow RLS policies to resolve operator identity without subquery joins into the `operators` or `devices` tables.

**Mechanism:** A Supabase Auth **custom access token hook** is registered as a Postgres function in Supabase Auth settings (Auth → Hooks → Custom Access Token). This hook fires on every token issuance — including login, refresh, and Admin API session creation. The hook function checks `devices` first (device users have no `operators` row), then `operators`:

```sql
CREATE OR REPLACE FUNCTION custom_access_token_hook(event jsonb)
RETURNS jsonb LANGUAGE plpgsql SECURITY DEFINER AS $$
DECLARE
  claims jsonb := event->'claims';
  uid    uuid  := (event->>'user_id')::uuid;
  op_id  uuid;
  dev_id uuid;
BEGIN
  SELECT operator_id, id INTO op_id, dev_id FROM devices WHERE user_id = uid;
  IF op_id IS NOT NULL THEN
    claims := jsonb_set(claims, '{operator_id}', to_jsonb(op_id));
    claims := jsonb_set(claims, '{device_id}',   to_jsonb(dev_id));
    RETURN jsonb_set(event, '{claims}', claims);
  END IF;
  SELECT id INTO op_id FROM operators WHERE user_id = uid;
  IF op_id IS NOT NULL THEN
    claims := jsonb_set(claims, '{operator_id}', to_jsonb(op_id));
  END IF;
  RETURN jsonb_set(event, '{claims}', claims);
END;
$$;
```

RLS policies reference `(auth.jwt()->>'operator_id')::uuid` and `(auth.jwt()->>'device_id')::uuid`. Dashboard tokens have `operator_id` but no `device_id`; device tokens have both. This distinction is used to restrict write access to dashboard users only.

### 3.4 RLS Policies

All tables have RLS enabled. Policies use JWT claims stamped by the custom access token hook (§3.4a). No policy in this system performs a subquery join into `devices` or `operators` to resolve the caller's operator.

**operators table:**
- SELECT: any authenticated user whose `operator_id` claim matches this row.
  ```sql
  USING (id = (auth.jwt()->>'operator_id')::uuid)
  ```
  This works for both dashboard users and device users — both have `operator_id` stamped.
- UPDATE: dashboard users only. Device tokens have a `device_id` claim; dashboard tokens do not.
  ```sql
  USING (
    id = (auth.jwt()->>'operator_id')::uuid
    AND (auth.jwt()->>'device_id') IS NULL
  )
  ```
- INSERT: service role only (signup trigger). DELETE: not allowed via policy.

**devices table:**
- SELECT: dashboard users see all devices for their operator; tablets see only their own row.
  ```sql
  USING (
    operator_id = (auth.jwt()->>'operator_id')::uuid
    OR id = (auth.jwt()->>'device_id')::uuid
  )
  ```
- UPDATE: two separate policies. Dashboard users update `display_name` and `status` for their operator's devices; tablets update `last_seen_at`, `app_version`, and `active_route_id` for their own row. Both use claim-based scoping.
- INSERT: service role only (`pair-device`).

**device_pairing_codes table:**
- All policies restricted to service role only (Edge Functions). No dashboard or tablet direct access.

**routes table:**
- SELECT: any authenticated user whose `operator_id` claim matches.
  ```sql
  USING (operator_id = (auth.jwt()->>'operator_id')::uuid)
  ```
- INSERT/UPDATE/DELETE: dashboard users only (no `device_id` claim).
  ```sql
  USING (
    operator_id = (auth.jwt()->>'operator_id')::uuid
    AND (auth.jwt()->>'device_id') IS NULL
  )
  ```

**route_stops table:**
- All operations: gated on the parent route's `operator_id`. A subquery into `routes` (indexed on PK) is unavoidable since `route_stops` does not carry `operator_id` directly; this is an O(1) lookup, not a full join.
  ```sql
  USING (
    (SELECT operator_id FROM routes WHERE id = route_id)
      = (auth.jwt()->>'operator_id')::uuid
  )
  ```
- INSERT/UPDATE/DELETE: additionally require no `device_id` claim (dashboard only).

**naptan_stations table:**
- SELECT: any authenticated user (dashboard or tablet).
- No INSERT/UPDATE/DELETE from clients; service-role-only.

### 3.5 Server-Side Timestamp Trigger

A trigger on the `routes` table sets `updated_at = now()` on every INSERT and UPDATE. This makes sync timestamps server-authoritative and immune to device clock drift.

```sql
CREATE OR REPLACE FUNCTION trigger_set_timestamp()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER routes_updated_at
  BEFORE INSERT OR UPDATE ON routes
  FOR EACH ROW
  EXECUTE FUNCTION trigger_set_timestamp();
```

The trigger applies only to `routes`. `route_stops` does not have its own timestamp because stops sync as a complete set with their parent route — modifying stops causes a route update, which triggers the route's timestamp.

---

## 4. Supabase Edge Functions

Edge Functions encapsulate operations that require service-role access or non-trivial logic that shouldn't live in the client.

### 4.1 generate-pairing-code

**Caller:** dashboard (authenticated operator)
**Purpose:** create a single-use pairing code for tablet pairing

**Input:** none (operator identity derived from JWT)

**Behaviour:**
1. Verify caller's JWT and resolve to an operator via the `operator_id` JWT claim (§3.4a).
2. Check `status = 'active'`; reject otherwise.
3. Generate a cryptographically random 6-digit numeric code; retry on collision with existing unexpired codes (checked against the `code` column, which has a `UNIQUE` constraint).
4. Insert into `device_pairing_codes` with `id = gen_random_uuid()`, `code` = the generated 6-digit value, `expires_at = now() + interval '10 minutes'`.
5. Return `{ code, expires_at }`.

### 4.2 pair-device

**Caller:** tablet (unauthenticated; calls with anon key)
**Purpose:** consume a pairing code, create a device record, return a session JWT and device secret

**Input:** `{ code: string, android_id: string }`

**Behaviour:**
1. Look up the pairing code by the `code` column (not by PK); reject if not found, expired, or already used.
2. Reject if the operator linked to the code has `status != 'active'`.
3. Create a new anonymous Supabase Auth user via the Admin API, setting `app_metadata: { role: 'device', operator_id: '<UUID>' }` (device_id is added in step 8 once the devices row UUID is known).
4. Generate a cryptographically random 256-bit device secret: `crypto.getRandomValues(new Uint8Array(32))`, base64url-encoded (43 chars).
5. Compute the SHA-256 hash of the plaintext secret (hex-encoded string).
6. **Count existing devices for this operator first.** Query `SELECT count(*) FROM devices WHERE operator_id = $op_id` and hold the result as `existing_count`. This must be done **before** the INSERT in step 7 — querying the count *after* the insert would always return ≥ 1 and silently flip every first-paired device's `audio_enabled` to `false`.
7. Insert a `devices` row in a single statement with: the new `user_id`, `operator_id` from the code, the `android_id` from the request, the `device_secret_hash`, default `display_name = "New Device #N"` where `N = existing_count + 1`, and **`audio_enabled = (existing_count = 0)`** — true for the first tablet paired to this operator, false for every subsequent tablet. The column's DEFAULT of `true` (§2.2) is overridden by this explicit value on every insert; no post-insert UPDATE is performed.
8. Update the anonymous user's `app_metadata` to add `device_id` (the UUID of the newly created devices row).
9. Mark the pairing code as used (`used_at = now()`).
10. Create a session for the anonymous user via the Admin API. The custom access token hook (§3.4a) fires automatically and stamps `operator_id` and `device_id` claims.
11. Return `{ access_token, refresh_token, device_id, operator_id, operator_name, device_secret }`. The `device_secret` is the plaintext value — included in this response only. The server does not log or retain it after the response is sent.

**Ordering guarantee (no race on first session).** Steps 7–9 (`devices` INSERT, `app_metadata` update, pairing-code mark-as-used) execute and commit before step 10 (`auth.admin.createSession`). The session-creation API call is sequenced strictly after the row writes — `pair-device` is plain procedural code, not a transaction-spanning RPC. The `custom_access_token_hook` (§3.4a) therefore always sees a committed `devices` row when it fires for the newly-created device's session, and the JWT carries both `operator_id` and `device_id` claims from the very first issuance. The previously-flagged race (hook fires before the devices row is visible, returning a JWT without `device_id`) cannot occur. If a failure interrupts the flow between step 7 and step 10, the orphaned `devices` row is harmless — it has no associated session, no tablet ever sees it, and it will be reaped by the 60-day auto-deregistration job (§10) because `last_seen_at` will never advance past its INSERT-time default.

Failure modes are handled cleanly: invalid code returns a clear error message; transient errors trigger a retry on the tablet.

**Integration-test requirement (audio_enabled regression).** A test must pair the very first device for a freshly-created operator and assert that the resulting `devices.audio_enabled = true`. This regression-tests the count-before-insert ordering — a naïve `SELECT count(*) … WHERE operator_id = $1` issued *after* the insert would always return ≥ 1 and silently flip every first device to `audio_enabled = false`, violating the §5.2 PRD claim that single-tablet operators require no configuration.

**Rate limiting:** To prevent brute-force enumeration of the 6-digit pairing code space, `pair-device` enforces:
- **Per-IP:** max 10 attempts per 10-minute window. Tracked in the `rate_limit_attempts` table (§2.9) — each attempt inserts a row with `endpoint = 'pair-device'`, `key = 'ip:' || <caller_ip>`, and `succeeded` set per the outcome. The window check is `SELECT count(*) FROM rate_limit_attempts WHERE endpoint = 'pair-device' AND key = 'ip:' || $ip AND attempted_at > now() - interval '10 minutes' AND NOT succeeded`.
- **Exponential backoff:** after 3 consecutive failures from the same IP within 60 seconds, subsequent attempts receive HTTP 429 with a `Retry-After` header. Back-off intervals: 5 s, 15 s, 60 s for attempts 4, 5, 6+.
- Failures against valid-but-already-used or expired codes count toward the limit; failures against genuinely invalid codes count at double weight (implemented by inserting two `rate_limit_attempts` rows for that class).
- Rate limit state resets when a new 10-minute window begins (older rows fall out of the windowed `count(*)` automatically; the §2.9 24-hour cleanup eventually deletes them).

### 4.3 recover-device

**Caller:** tablet (unauthenticated; calls with anon key)
**Purpose:** silently re-issue a session for an existing device after refresh token expiry

**Input:** `{ android_id: string, device_secret: string }`

**Behaviour:**
1. Look up the `devices` row by `android_id`. Reject with HTTP 404 if not found.
2. Compute the SHA-256 hash of the provided `device_secret`. Compare against the stored `device_secret_hash` using a constant-time comparison. Reject with HTTP 401 if the hashes do not match.
3. Reject if `devices.activation_state = 'inactive'` or if the linked operator has `status != 'active'`.
4. Issue a fresh session for the device's `user_id` via the Admin API. The custom access token hook (§3.4a) fires automatically and stamps `operator_id` and `device_id` claims.
5. Return `{ access_token, refresh_token, device_id, operator_id }`.

**Failure classification (consumed by the tablet — see PRD FR-AT-04).** The tablet treats `recover-device` outcomes as either *terminal* (HTTP 404 device-row-missing; HTTP 401 secret-mismatch; HTTP response indicating `activation_state = 'inactive'` or operator-disabled) or *transient* (HTTP 5xx, network errors, HTTP 429 rate-limit, anything else). On terminal failures the tablet wipes cached credentials and returns to first-run pairing; on transient failures the tablet retains its cached JWT/refresh token/device secret and retries on the next sync. The endpoint itself returns the same shape regardless — the classification lives in the caller. This split exists to keep a working in-service tablet operating through transient Supabase/FCM/network blips rather than dropping it into a pairing-code-required state on a bus mid-journey.

**Rate limiting:** `recover-device` is an unauthenticated endpoint. To limit brute-force attempts against the device secret:
- **Per-Android-ID:** max 5 attempts per hour. After 3 consecutive failures (wrong secret or inactive device), lock out that Android ID for 15 minutes. Tracked in `rate_limit_attempts` (§2.9) with `endpoint = 'recover-device'` and `key = 'android:' || <android_id>`.
- **Per-IP:** max 20 attempts per hour across all Android IDs from that IP. Tracked in the same table with `key = 'ip:' || <caller_ip>`.
- HTTP 429 with `Retry-After` on breach.
- A legitimate recovery (tablet returning after months offline) succeeds on the first attempt; these limits only prevent systematic brute-force.

### 4.4 replace_route_with_stops

**Caller:** dashboard (authenticated operator); also reserved for future tablet use
**Type:** PostgreSQL RPC (function, not Edge Function — runs inside the database)
**Purpose:** atomically upsert a route and replace its entire stop list in a single transaction

**Input:** `{ p_route: route_input, p_stops: jsonb }`
where `route_input` is a composite type matching the routes table columns. Each stop object in the array mirrors the `route_stops` table columns, including `proximity_radius_meters` and `segment_type`.

**Behaviour (in a single transaction):**
1. UPSERT the route row (insert or update by id).
2. DELETE all existing route_stops for that route_id.
3. INSERT the stops from the JSON array.
4. Compute `stops_content_hash` from the just-inserted stops (canonical form: for each stop in `stop_order` ascending, the string `"{naptan_id}|{stop_order}"`; the strings joined by newline; UTF-8 bytes hashed with SHA-256; lowercase hex output). `UPDATE routes SET stops_content_hash = $1 WHERE id = $route_id`.

The server's `updated_at` trigger fires on the route UPSERT (step 1) and again on the hash UPDATE (step 4), but the cursor is server-authoritative so the final `updated_at` is still well-defined within the transaction.

**stops_content_hash semantics.** `updated_at` changes on every save (it is the sync cursor — and that is correct). `stops_content_hash` only changes when the structural stop list actually changes (stop added/removed/reordered, or a stop's `naptan_id` changes). A route metadata edit (renaming the route, changing `route_number`, changing `direction`, toggling `audio_enabled` elsewhere) writes the same stops, hashes to the same value, and leaves `stops_content_hash` unchanged. PRD FR-WD-12 relies on this property to fire the return-route divergence warning only on structural drift, avoiding warning fatigue.

This RPC is what the dashboard calls when saving a route.

**Audio rendering:** After calling this RPC successfully, the dashboard calls the
`enqueue-render-job` Edge Function (§4.6) which enqueues a `render-route-audio` job onto the
pg_boss queue. The dashboard does not call any synchronous render API and does not wait for
the render to complete. The route appears in the route list immediately with
`audio_render_status = 'pending'`; the status flips to `ok` (or `failed`) when the
`audio-render-worker` processes the job. The dashboard does **not** fire any FCM push on save —
FCM dispatch is the render worker's responsibility on successful completion (§4.6 and §8.4).

**Audio rendering on return-route generation and re-generation (PRD FR-WD-12).** The
dashboard's "Generate return route" Server Action creates the return route entity by calling
`replace_route_with_stops` for the new route, then **must explicitly call `enqueue-render-job`
for the new return route's ID at its newly-stamped `updated_at`** — exactly as the route-save
path does for the source route. The same obligation applies on "Re-generate return route": after
`replace_route_with_stops` has updated the existing return route's stops, the Server Action
must call `enqueue-render-job` for that route. Without the explicit enqueue, the return route
would carry `audio_render_status = 'pending'` indefinitely and its journey-start gate would
remain closed. Server-side deduplication in `enqueue-render-job` (§4.6) guarantees that a
duplicate enqueue from a quick double-save is harmless.

### 4.6 Audio Render Job (pg_boss queue + scheduled worker)

Audio rendering runs as a **pg_boss-backed job**, not as a synchronously-invoked Edge Function.
This section specifies the queue, the enqueue path, the worker, and the per-job processing
algorithm. The job replaces the previous `render-route-audio` Edge Function (removed in v3.8);
the *output* is unchanged (pre-rendered WAV files (LINEAR16 PCM) in Supabase Storage — see §2.8)
but the production mechanism is asynchronous, retryable, idempotent, and observable.

**Why pg_boss.** A Postgres-backed job queue gives us durability, retry, and backoff with no
new infrastructure: pg_boss creates its own tables inside the existing Supabase Postgres
database. There is no separate queue service to deploy or pay for. The trade-off — the worker
runs as a scheduled Edge Function rather than a long-running consumer — is acceptable for the
initial release's expected queue depth (a single operator may save a handful of routes per
week; even at 100 operators that is tens of jobs per day).

#### INVIOLABLE RULE — pg_boss configuration in Edge Function isolates

The `audio-render-worker` Edge Function (and the `pgboss-maintain` function below, and any
other Edge Function that instantiates pg_boss) **MUST** construct pg_boss with
`{ supervise: false, schedule: false }`:

    const boss = new PgBoss({
      connectionString: Deno.env.get('SUPABASE_DB_URL'),
      supervise: false,
      schedule: false,
    });

This is non-negotiable. pg_boss's defaults (`supervise: true`, `schedule: true`) start an
in-process supervision loop and a scheduling timer that both assume a long-lived consumer.
In a short-lived Deno isolate the timers fire after the request handler has returned, race
the runtime's wall-clock termination, leak handles, and can leave partially-committed work.
With `{ supervise: false, schedule: false }` pg_boss runs as a pure DB-driven state machine:
`fetch`, `complete`, `fail`, and `maintain` are explicit operations rather than
side-effects of a supervisor loop. The retry state machine still works correctly across
isolates because it is database-driven (`pgboss.job.state` + `retry_count`).

This was verified empirically in the round-3 spike (`spike-records/round-3/findings-pg_boss.md`).
The spike measured 50 ms cold start, 8–14 ms warm, ~5 ms `boss.maintain`, and the full
retry lifecycle across three short-lived invocations. The defaults produced exactly the
"leaked timer" symptoms the rule prevents.

#### Maintenance: `pgboss-maintain` scheduled Edge Function

Because `supervise: false` disarms pg_boss's internal maintenance loop, archive/expiry and
dead-job-recovery duties must be driven externally. A small scheduled Edge Function
`pgboss-maintain` runs `await boss.maintain()` on `cron: '0 * * * *'` (top of every hour):

    // pgboss-maintain Edge Function — runs at 0 * * * *
    const boss = new PgBoss({ connectionString: ..., supervise: false, schedule: false });
    await boss.start();
    await boss.maintain();
    await boss.stop();

The function does ~5 ms of real work per tick (mostly idle wall-clock) and ~50 ms on a cold
start. It shares the scheduled-function infrastructure that already runs the daily 03:00 UTC
cleanup (`audio-cleanup-worker`, `rate_limit_attempts` cleanup, 60-day device
auto-deregistration), but on its own hourly schedule rather than colocated with the daily
job — `boss.maintain()` is the only call it makes, and the hourly cadence bounds active-job
visibility timeouts at ~1 hour.

#### Installation and configuration

- pg_boss is installed by running `await boss.start()` once from a one-off bootstrap Edge
  Function (`pgboss-install`). This creates the `pgboss` schema and the tables
  `pgboss.job`, `pgboss.archive`, `pgboss.schedule`, `pgboss.subscription`, `pgboss.version`.
  We do not define a custom job table — pg_boss owns its schema and handles its own internal
  migrations across library upgrades.
- Queue name: `render-route-audio`.
- Retry policy: `retryLimit: 5`, `retryDelay: 30` (seconds), `retryBackoff: true`. pg_boss's
  native exponential backoff yields retries at approximately 30s, 60s, 120s, 240s, 480s
  after the previous failure.
- Job retention: `retentionDays: 7` (completed and terminally-failed jobs archive after 7 days).

#### Enqueue path: `enqueue-render-job` Edge Function

**Caller:** dashboard (after a successful `replace_route_with_stops`).
**Purpose:** push a job onto the `render-route-audio` pg_boss queue.

**Input:** `{ route_id: string, route_version: number }` where `route_version` is the freshly
stamped `routes.updated_at` expressed as epoch millis (the dashboard reads this back from the
`replace_route_with_stops` response).

**Behaviour:**
1. Verify caller's JWT and resolve to an operator via the `operator_id` claim. Confirm the
   route belongs to that operator (RLS-protected SELECT on `routes`).
2. Connect to pg_boss via the Supabase Postgres connection (`SUPABASE_DB_URL`). Instantiate
   pg_boss with `{ supervise: false, schedule: false }` per the inviolable rule above.
3. **Server-side deduplication check (authoritative).** Query pg_boss for any existing job
   on the `render-route-audio` queue with `state IN ('created', 'active')` whose payload
   matches `{ route_id, route_version }` exactly. If one exists, return that job's
   `{ job_id, dedup: true }` without enqueueing a duplicate. The check is a single SQL
   query against `pgboss.job` filtered by queue name, state, and `data ->> 'route_id'` /
   `(data ->> 'route_version')::bigint`. This is the **authoritative** dedup for FR-WD-21
   (the client-side 60-second disable is UX only). Suppressing the duplicate enqueue here
   prevents the redundant-FCM-dispatch behaviour that round-3 finding 20 named (two clicks
   would otherwise enqueue two jobs; both pass the staleness check, both find files
   already present from the first run, both call `boss.complete`, and both dispatch FCM
   on completion). The per-job staleness check (step 1 of "Per-job processing" below) is
   still a correctness safety net — it handles the case where dedup is bypassed because
   the first job has already moved out of `(created, active)` into `completed`.
4. Call `boss.send('render-route-audio', { route_id, route_version })`.
5. Return `{ job_id, dedup: false }`.

Direct client-side enqueue is avoided because pg_boss writes require Postgres credentials that
must not be exposed to the browser. `enqueue-render-job` is deliberately thin — it does not
read the route data itself; that is the worker's job.

The dashboard also offers a "Re-render audio" action per route (PRD FR-WD-21) which calls
`enqueue-render-job` with the route's current `updated_at` as `route_version` — useful for
recovering from terminal failures without modifying route data. The dedup check above is the
mechanism that makes rapid double-clicks idempotent.

#### Worker: `audio-render-worker` Edge Function

A Supabase Edge Function invoked **every minute** by Supabase scheduled functions
(`cron: '* * * * *'`). Each invocation does:

1. Connect to pg_boss using `SUPABASE_DB_URL`.
2. `boss.fetch('render-route-audio', batchSize)` — pull up to `batchSize = 5` ready jobs.
3. Process each fetched job synchronously within the invocation (see "Per-job processing"
   below).
4. On per-job success: `boss.complete(jobId)`. On per-job failure: `boss.fail(jobId, err)` —
   pg_boss schedules the retry with backoff.
5. Edge Function execution time on Supabase is bounded (typically 60s for hosted). The worker
   exits when 50s have elapsed in the current invocation even if jobs remain — those jobs
   stay in the queue and are picked up by the next scheduled run. The 10s safety margin
   allows in-flight `boss.complete` / `boss.fail` calls to commit cleanly.

The scheduled-function worker is the simpler initial implementation. If queue depth grows to
the point that one-minute-granularity polling and batch-of-5 throughput become a bottleneck,
the worker can be promoted to a dedicated long-running consumer (e.g., a Fly.io machine) with
no change to the queue, the job payload, or the per-job processing algorithm. The current
design defers that complexity until it is justified by load.

#### Per-job processing

**Job payload:** `{ route_id: string, route_version: number }`.

1. **Staleness check.** Load the `routes` row by `route_id`. If `extract(epoch from
   routes.updated_at) * 1000 ≠ route_version`, a newer save has happened since this job was
   enqueued. Mark the job complete (`boss.complete`) without rendering — the newer save has
   enqueued its own job which will produce the correct audio. This prevents wasted Google TTS
   spend on superseded versions.
2. Load all `route_stops` rows for `route_id`, ordered by `stop_order`.
3. Resolve `operator_id` from the route row (for Storage paths).
4. Compute the announcement text:
   `"This bus is the [route.name] service to [last_stop.stop_name]."`. Compute its SHA-256 hex
   hash as `would_be_announcement_hash`.
5. For each stop, compute the per-stop text `"Next stop: [stop.stop_name]."` and its SHA-256
   hex as that stop's `would_be_content_hash`.
6. **Differential re-render check (announcement and each stop):**
   - Define `target_path = {operator_id}/{route_id}/{route_version}/<file>.wav`.
   - If the stored hash on the database row (`routes.audio_announcement_hash` or
     `route_stops.audio_content_hash`) equals the `would_be_*_hash`, the text is unchanged
     since the last successful render. The worker does **not** call Google TTS.
   - The worker then checks Storage for the file at `target_path`. If present, nothing to do.
     If absent (the typical case — this is a new `route_version` and the version path is
     fresh), the worker locates the file at the **previous version's path** (the most-recent
     `route_version' < route_version` for this route that contains a file matching the same
     hash) and uses Storage's server-side `copy` to place a copy at `target_path`. This makes
     a direction-label-only edit (which changes nothing about any announcement text)
     genuinely free of TTS cost — the worker only copies files.
   - If no previous-version file exists with a matching hash, fall through to the render path.
7. **Render path (for any announcement or stop whose hash changed or has no copyable prior):**
   - Call Google Cloud Text-to-Speech `synthesizeSpeech` with:
     - `voice: { languageCode: 'en-GB', name: 'en-GB-Neural2-B' }`
     - `audioConfig: { audioEncoding: 'LINEAR16', sampleRateHertz: 24000 }`
       (24 kHz is the voice's native rate, verified in the round-3 spike — see
       `spike-records/round-3/findings-tts-frequency.md`; LINEAR16 is required end-to-end
       to preserve the Reg 13(4)-verified spectrum — see Compliance Mapping Matrix.)
   - **The voice is locked to `en-GB-Neural2-B`.** It is not configurable per operator, per
     route, or per deployment. Changing the voice requires re-running the Reg 13(4)
     frequency verification (Compliance Mapping Matrix) and is therefore a deliberate
     re-planning event, not a runtime knob. This is an inviolable rule alongside the
     alert-chime sequence.
   - Upload the resulting LINEAR16 PCM bytes wrapped in a canonical PCM WAV container
     (44-byte header) to Storage at `target_path` using the service-role Supabase client.
     The Google TTS REST endpoint returns raw LINEAR16 samples; the worker prepends the
     standard WAV/RIFF header (mono, 24 kHz, 16-bit signed little-endian) before upload.
   - On a successful upload, update the corresponding hash column atomically in a single
     SQL statement:
     - For the announcement: `UPDATE routes SET audio_announcement_hash = $1 WHERE id = $2`.
     - For a stop: `UPDATE route_stops SET audio_content_hash = $1 WHERE id = $2`.
8. **Partial-failure semantics.** The worker processes the announcement and stops
   sequentially. If any single step fails (TTS API error, Storage upload error, transient
   network error), the worker calls `boss.fail(jobId, err)`; pg_boss schedules a retry per the
   backoff policy. **Already-completed steps are durable**: their Storage files and updated
   hash columns persist across the retry. On retry, the differential re-render check (step 6)
   short-circuits every step that already succeeded, so processing resumes at the failed
   step. There is no explicit "where did we stop" cursor — the content-hash bookkeeping is
   the resumption mechanism.
9. **Terminal failure.** After `retryLimit` retries (5) are exhausted, pg_boss marks the job
   permanently failed. The worker's failure handler runs and:
   - `UPDATE routes SET audio_render_status = 'failed', audio_render_error = $err_message
     WHERE id = $route_id`. The error message captured is the last failure detail (truncated
     to a reasonable length).
   - **Does not fire FCM.** Tablets are not notified of a failure. The route's
     `audio_render_status` reaches tablets on their next regular sync (the column rides along
     in the `get_routes_since` response, §4.5) and the tablet skips audio download for that
     route (§7.4). The "Audio not ready" indicator on the tablet (§6.4) continues to gate
     journey starts.
   - The dashboard surfaces the failure on the route list (PRD FR-WD-13) with a "Re-render
     audio" action (PRD FR-WD-21) to re-enqueue.
10. **On successful completion of the whole job** (announcement + all stops either rendered,
    skipped-by-hash, or copied-from-previous-version):
    - `UPDATE routes SET audio_render_status = 'ok', audio_render_error = NULL WHERE
      id = $route_id`.
    - **Fire FCM push** by calling the route-change FCM dispatcher (§7.6) for this operator's
      active devices. This is the only path that fires FCM. The render-then-FCM ordering
      guarantees that any tablet receiving the push will find both the route data and the
      version-keyed audio files available for download.

#### Fixed announcements are not rendered here

Termination, hail-and-ride start/end, diversion start/end, and the alert chime are bundled in
the APK (§6.3) and do not vary by route. The audio-render-worker only renders route-specific
files. The bundled files are rendered once during development using the **same locked voice
(`en-GB-Neural2-B`)** and shipped in the APK to ensure voice consistency between fixed and
route-specific audio.

#### Cost discipline

- Google TTS Neural2 voice pricing is approximately $16 per 1M characters (as of late 2025).
- A 10-stop route is ~250 characters of TTS input; a full first-time render costs ~$0.004.
- Editing one stop name re-renders one file (~25 characters) for ~$0.0004; the
  differential hash check ensures unchanged stops cost nothing.
- The 5-retry cap with backoff bounds the worst-case per-job retry cost.
- The differential hash check + previous-version copy together ensure that a direction-label
  edit, which changes no announcement text, calls Google TTS zero times.
- No explicit cost tracking surface is required for the initial release; the Google Cloud
  billing dashboard and pg_boss queue depth provide adequate visibility.

### 4.7 Storage Cleanup Job (audio-cleanup-worker)

A separate scheduled Edge Function `audio-cleanup-worker` runs daily (`cron: '0 3 * * *'`,
03:00 UTC). For each route in the database:

1. List Storage entries under `{operator_id}/{route_id}/`. Each immediate child folder is a
   `route_version`.
2. Sort versions descending (largest epoch-millis first).
3. Keep the **three most-recent** versions. Recursively delete every older version path.

**Why three versions.** Three-version retention gives a tablet a recovery window across the
common rapid-iteration pattern an operator produces in practice: a "save, notice mistake,
save again, notice another mistake, save once more" cycle within a single cleanup window
produces 3 versions. With **two-version** retention (the v3.8 policy) the same scenario
left a tablet that synced an intermediate version stranded: a tablet that synced version N+1
at 02:35 — moments before the operator went on to save N+2 at 02:45 and N+3 at 02:55 — would
find at the 03:00 UTC cleanup that only N+2 and N+3 survived; N+1's Storage path would 404
until the tablet's next sync. Three versions absorbs the typical iteration depth without
materially increasing Storage cost (the LINEAR16/WAV audio for one route is ≈ 2.6 MB per
version — see §2.8; three versions ≈ 8 MB per route, still small relative to other Storage
usage).

**Named residual assumption.** An operator iterating more than 3 versions of a route
within a single cleanup cycle (currently 24 hours, 03:00 UTC) may produce transient
`AUDIO_FILE_MISSING` on tablets that synced an intermediate version. The condition
**self-heals on the next sync** (the tablet pulls the new `route_version` from metadata
and downloads from the now-current Storage path; the journey-start gate then opens). The
03:00 UTC cleanup window is chosen partly because route editing concentrates during the
working day; running cleanup in a low-activity window minimises the chance of catching a
mid-iteration tablet. Acceptable behaviour for the initial release; if rapid iteration
ever becomes routine, the retention count can be lifted or the cleanup cadence relaxed.

The cleanup job is idempotent — running it more than once a day is harmless because each run
recomputes the keep-set independently. Failure within a day is logged but not retried inside
the same day; the next day's run picks up the missed work.

**Soft-deleted routes.** When a route is soft-deleted (`routes.is_deleted = true`), the
three-version retention still applies — all surviving versions remain in place. A future
cleanup pass may delete all audio versions for soft-deleted routes after a longer retention
window; the initial release leaves them in place, which is simpler and still bounded by the
three-version cap.

**Manual hard-delete of a route** (a deliberate administrative action; the initial release
does **not** perform scheduled hard deletes, see §10) removes all versions for that route in
the same pass that removes the row.

### 4.5 get_routes_since

**Caller:** tablet
**Type:** PostgreSQL RPC
**Purpose:** download all routes (including stops) updated since a cursor timestamp

**Input:** `{ p_last_ts: bigint }` (epoch millis)

**Behaviour:**
1. Capture `current_timestamp` from the transaction at function start (this becomes the next cursor).
2. Query routes where `operator_id = (auth.jwt()->>'operator_id')::uuid` (resolved from the JWT claim — see §3.4a) AND `updated_at > to_timestamp(p_last_ts / 1000.0)`.
3. For each matching route, attach its full route_stops list as a JSON array.
4. Return `{ server_now: bigint, routes: jsonb }` where `server_now` is the captured transaction timestamp converted to epoch millis.

The tablet stores `server_now` as the next cursor. Using the server's transaction timestamp (not the max `updated_at` from the batch) prevents edge cases where new writes happen during the read.

---

## 5. Room (Android) Schema — On-Device

These tables live on the tablet in an SQLite database managed by Android Room. The tablet reads exclusively from these tables during operation.

### 5.1 routes (Synced)

Local mirror of the operator's routes from Supabase.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | TEXT | PK | UUID as string, matches Supabase routes.id |
| operator_id | TEXT | NOT NULL | Matches Supabase |
| name | TEXT | NOT NULL | Route display name |
| route_number | TEXT | NULLABLE | Optional route number |
| direction | TEXT | NULLABLE | Direction label |
| return_route_id | TEXT | NULLABLE | Links to return route |
| updated_at_utc | LONG | NOT NULL | Epoch millis of server timestamp. Also used as the `route_version` path segment for audio downloads (§7.2 step 7, §2.8). |
| is_deleted | BOOLEAN | NOT NULL | Soft delete flag from Supabase. When true, hidden from route list. |
| audio_render_status | TEXT | NOT NULL, DEFAULT 'pending' | Mirror of `routes.audio_render_status` from Supabase (`pending` / `ok` / `failed`). Read by the audio-download step (§7.4) — only `ok` routes attempt to download audio. Read by the journey-start gate (§6.4) which requires `ok` AND all files present locally. |
| pending_deletion | BOOLEAN | NOT NULL, DEFAULT false | Local-only: true if sync pulled a remotely deleted route that is currently active in a journey. Route stays usable for that journey but hidden from the route list. Cleanup sets is_deleted = true when the journey ends. |
| last_used_at | LONG | NULLABLE | Local-only: epoch millis of last journey start with this route. Used for sorting. Not synced. |

### 5.2 route_stops (Synced)

Local mirror of stops. Replaced entirely during sync.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | TEXT | PK | UUID as string, matches Supabase |
| route_id | TEXT | NOT NULL, FK → routes(id) ON DELETE CASCADE | Parent route |
| naptan_id | TEXT | NULLABLE | NaPTAN identifier |
| stop_name | TEXT | NOT NULL | Display name |
| crs_code | TEXT | NULLABLE | 3-letter station code |
| latitude | DOUBLE | NOT NULL | WGS84 latitude |
| longitude | DOUBLE | NOT NULL | WGS84 longitude |
| stop_order | INTEGER | NOT NULL | Sequence position |
| is_custom | BOOLEAN | NOT NULL | False in the initial release |
| proximity_radius_meters | INTEGER | NOT NULL | Per-stop proximity radius in metres. Synced from Supabase. Read by the GPS detection service on each position update. |
| segment_type | TEXT | NOT NULL | Segment classification: 'scheduled' or 'hail_and_ride'. Synced from Supabase. Read by the GPS detection service to determine announcement behaviour and tube-map rendering at each stop. |

**Index:** `route_id`.

### 5.4 journey_events (Local Only)

Diagnostic event log. Not synced. Auto-pruned after 30 days.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | LONG | PK, AUTO-INCREMENT | Local auto-incrementing ID |
| event_type | TEXT | NOT NULL | JOURNEY_START, JOURNEY_END, JOURNEY_AUTO_CLEARED, STOP_ANNOUNCED, STOP_PASSED_WITHOUT_DETECTION, STOP_SKIPPED, HAIL_AND_RIDE_SECTION_STARTED, HAIL_AND_RIDE_SECTION_ENDED, DIVERSION_STARTED, DIVERSION_ENDED, DIVERSION_REPLAYED, GPS_LOST, GPS_REGAINED, AUDIO_DISCONNECTED, AUDIO_RECONNECTED, AUDIO_FILE_MISSING, AUDIO_PLAYBACK_ERROR, GMS_OVERRIDE_ACKNOWLEDGED, CLOCK_DRIFT, SYNC_SUCCESS, SYNC_FAILURE, APP_ERROR. `JOURNEY_AUTO_CLEARED` is logged when journey state is cleared on recovery due to staleness (PRD FR-AT-18; `detail` = `'stale_age'` or `'stale_no_events'`). `DIVERSION_REPLAYED` is logged when the diversion-start announcement is replayed on journey recovery (PRD FR-AT-18; `detail` = `'recovery'`). `GMS_OVERRIDE_ACKNOWLEDGED` is logged when the user dismisses the GMS unavailability block (PRD FR-AT-67; `detail` = GMS status code, e.g. `'SERVICE_MISSING'`). |
| route_id | TEXT | NULLABLE | Route ID if relevant |
| stop_name | TEXT | NULLABLE | Stop name if relevant |
| trigger_method | TEXT | NULLABLE | "GPS" (normal proximity entry), "GPS_INFERRED" (stop auto-advanced via two-stop look-ahead without direct proximity entry), or "MANUAL" for stop announcements. `STOP_SKIPPED` events use trigger_method = "DRIVER_DIVERSION". `HAIL_AND_RIDE_SECTION_STARTED` and `HAIL_AND_RIDE_SECTION_ENDED` use trigger_method = "GPS" (automatic boundary detection) or "MANUAL" (driver fallback). |
| detail | TEXT | NULLABLE | Additional context (e.g., error message) |
| timestamp_utc | LONG | NOT NULL | Epoch millis, UTC |

### 5.5 journey_state (Local Only)

Persists the active journey state for crash recovery. Single-row table.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | INTEGER | PK, hardcoded to 1 | Enforced via `@PrimaryKey(autoGenerate = false)` and `@Insert(onConflict = REPLACE)` in the DAO. Prevents duplicate-insert crashes. |
| route_id | TEXT | NOT NULL | The active route |
| current_stop_index | INTEGER | NOT NULL | 0-based index of the current/next stop |
| journey_started_at | LONG | NOT NULL | Epoch millis when the journey began. Read by the staleness check on app launch (see "Recovery staleness rules" below) — a journey older than `JOURNEY_STATE_MAX_AGE_HOURS` is auto-cleared rather than resumed. |
| is_active | BOOLEAN | NOT NULL | True during a journey. On app restart, if true, the journey is resumed **subject to the staleness check below** — a stale residue from a prior shift is auto-cleared, not resumed. |
| diversion_invoked_at_any_point | BOOLEAN | NOT NULL, DEFAULT false | **One-way latch within a journey.** Set `true` the first time the driver triggers the diversion-start action (FR-AT-25) during the active journey. **Never reset to `false` mid-journey** — even when the diversion is ended and `journey_skipped_stops` is cleared, this column remains `true` for the remainder of the journey. Reset to default `false` only when a new journey starts (a fresh row replaces the single row at journey start, with the rest of the columns populated for the new journey). Read at journey end by §7.8 to populate `journey_summary.diversion_invoked` correctly — covers the case where the diversion was started and then ended mid-journey (which clears `journey_skipped_stops` but leaves this latch true). |

**Note on diversion state.** Skipped stops for an active diversion are stored in the `journey_skipped_stops` table (§5.7) — that table is the working set the GPS state machine consults on every stop advance, and it is cleared on "Diversion end." The **fact that a diversion occurred at all during this journey** is captured by the `diversion_invoked_at_any_point` latch on this row. Together they support both fast indexed skip lookup (the table) and a correct journey-summary metric regardless of whether the diversion was still active at journey end (the latch).

**Recovery staleness rules (PRD FR-AT-18).** When the app launches and finds `is_active = true`, two thresholds gate the resume decision:

- `JOURNEY_STATE_MAX_AGE_HOURS = 8` — covers a full driving shift. A journey whose `journey_started_at` is more than 8 hours ago is by definition a stale residue from a prior day (the app was killed before journey-end and the tablet sat unused overnight).
- `JOURNEY_EVENT_RECENCY_THRESHOLD_HOURS = 1` — a healthy in-service tablet logs `STOP_ANNOUNCED`, `GPS_LOST`, or similar events frequently. A full hour with no `journey_events` row since `journey_started_at` strongly indicates the bus has stopped operating without a clean shutdown.

Recovery algorithm (see PRD FR-AT-18 for the full FR specification):

1. If `now() - journey_started_at > JOURNEY_STATE_MAX_AGE_HOURS`: clear `is_active = false`, clear `journey_skipped_stops`, log `JOURNEY_AUTO_CLEARED` with `detail = 'stale_age'`, return to route list.
2. Else if no `journey_events` row exists with `timestamp_utc >= journey_started_at`, or the most recent such row's `timestamp_utc` is older than `now() - JOURNEY_EVENT_RECENCY_THRESHOLD_HOURS`: clear as above with `detail = 'stale_no_events'`.
3. Else: resume the journey. If `journey_skipped_stops` is non-empty, replay the diversion start announcement before re-arming the GPS state machine (PRD FR-AT-18 diversion-replay step).

**Note on single-row design and multi-tablet future:** The single-row design (one active journey per device) is correct for the initial release. Each tablet is an independent device with its own journey state; there is no shared journey state between tablets in the initial release (see PRD §5.2). The PRD §5.2 gestures at a future "primary-secondary tablet linking with shared journey state" feature. That feature would require a fundamentally different architecture — journey state would need to be a Supabase Realtime-projected record rather than a local single-row table. That redesign is deliberately deferred. The single-row `journey_state` table is the correct, intentional design for the initial release scope.

### 5.6 sync_metadata (Local Only)

Tracks sync state. Single-row table.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | INTEGER | PK, hardcoded to 1 | Same single-row pattern as journey_state |
| last_sync_at | LONG | NULLABLE | Epoch millis of last successful sync completion |
| sync_status | TEXT | NOT NULL, DEFAULT 'never' | 'synced', 'syncing', 'failed', 'never' |
| last_server_timestamp | LONG | NOT NULL, DEFAULT 0 | Server transaction timestamp from the last successful download. Used as the sync cursor for `get_routes_since`. |
| pending_account_status | TEXT | NULLABLE, DEFAULT NULL | Captured operator account status when a sync occurs during an active journey (`journey_state.is_active = true`). Values: NULL (nothing captured / operator is active), `'pending'`, or `'suspended'`. Read at journey end (§7.8) to decide whether to route to the Account Status Screen (FR-AT-60). Cleared when a subsequent sync sees `status = 'active'`. See §7.4 "Mid-journey suspension grace." |

### 5.7 journey_skipped_stops (Local Only)

Transient list of stop indices skipped due to an active driver-initiated diversion. Not synced. Cleared at journey start and end.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | INTEGER | PK, AUTO-INCREMENT | Local auto-incrementing ID |
| stop_index | INTEGER | NOT NULL, UNIQUE | The 0-based `stop_order` index of the stop to skip. UNIQUE prevents duplicate entries for the same stop. |

**Implementation note:** When `journey_state.current_stop_index` advances to a value present in this table, the GPS state machine immediately increments the index again (repeatedly, until reaching a non-skipped index) without waiting for GPS proximity and without firing any announcement. A `STOP_SKIPPED` event is logged for each skipped stop. This table is emptied (`DELETE FROM journey_skipped_stops`) at journey start and on "Diversion end." Since there is only ever one active journey at a time, no journey ID foreign key is required.

### 5.8 journey_summaries_pending (Local Only)

Holds journey-summary rows queued for upload to Supabase `journey_summaries` (§2.6). Written at journey end (§7.8); rows are deleted after a successful upload. Unsynced rows survive app restarts and reboots.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | TEXT | PK | Locally-generated UUID. Used as the eventual Supabase `journey_summaries.id` so retries are idempotent. |
| device_id | TEXT | NOT NULL | This device's UUID (matches Supabase) |
| operator_id | TEXT | NOT NULL | Owning operator UUID |
| route_id | TEXT | NULLABLE | The route that was run. May be NULL if the route was locally removed before the summary was written |
| journey_started_at | LONG | NOT NULL | Epoch millis |
| journey_ended_at | LONG | NOT NULL | Epoch millis |
| stops_announced_count | INTEGER | NOT NULL, DEFAULT 0 | Count of STOP_ANNOUNCED events |
| stops_passed_without_detection_count | INTEGER | NOT NULL, DEFAULT 0 | Count of STOP_PASSED_WITHOUT_DETECTION events |
| manual_advances_count | INTEGER | NOT NULL, DEFAULT 0 | Count of STOP_ANNOUNCED with trigger_method = 'MANUAL' |
| gps_lost_events_count | INTEGER | NOT NULL, DEFAULT 0 | Count of GPS_LOST events |
| audio_failures_count | INTEGER | NOT NULL, DEFAULT 0 | Count of AUDIO_FILE_MISSING + AUDIO_PLAYBACK_ERROR events |
| diversion_invoked | INTEGER | NOT NULL, DEFAULT 0 | 0/1 (SQLite has no native BOOLEAN). True if journey_skipped_stops was non-empty at any point |
| needs_upload | INTEGER | NOT NULL, DEFAULT 1 | 0/1. True until the row has been successfully inserted into Supabase `journey_summaries`. After a successful upload, the row is deleted from this table rather than flipped — `needs_upload` exists as a state flag for the upload step to retry on, and to make resumable upload observable in diagnostic queries. |
| last_upload_attempt_at | LONG | NULLABLE | Epoch millis of last upload attempt, if any. Diagnostic only. |

**Index:** `needs_upload` (for the upload step's lookup of pending rows).

**Single legitimate use of upload-sync metadata.** The v3.6 changelog removed `needs_upload` placeholders from `routes` (§5.1) and removed the upload-sync algorithm wholesale from §7. That removal targeted a speculative future *route-upload* path that the initial release does not need; routes remain read-only on the tablet. The `journey_summaries_pending` table is a deliberate, scoped exception — it exists for one specific, narrow purpose (uploading anonymous per-journey count metrics at journey end), and `needs_upload` here is **not** a return of generic upload-sync scaffolding. Any future feature wanting per-entity upload metadata should make its case on its own merits; the v3.6 removal still stands for everything else.

### 5.9 device_state (Local Only)

A single-row Room table holding device-level state that the tablet maintains locally between syncs. Same single-row pattern as `journey_state` (§5.5) and `sync_metadata` (§5.6) — `@PrimaryKey(autoGenerate = false)` with `id` hardcoded to `1`, and `@Insert(onConflict = OnConflictStrategy.REPLACE)` in the DAO. Designed for forward-compatibility: additional device-level cached state (e.g. a future operator-account-status mirror, per-device feature flags) should live here rather than spawning yet more single-row tables.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | INTEGER | PK, hardcoded to 1 | Single-row enforcement (see pattern above). |
| audio_enabled | BOOLEAN | NOT NULL | Local cached copy of this device's `devices.audio_enabled` value (§2.2). Written at the end of every successful operator-row sync by §7.2 step 8. Read by: the audio downloader (§7.2 step 7 `audio_enabled` guard), the audio playback engine (FR-AT-28), and the §7.2 step 8 flip-detection logic that compares the freshly synced value against this cached value. Replaces the previously-dangling reference to "the most recently synced `devices` row" — that row exists only in Supabase; the tablet needs a local copy to read at every audio-decision point. |
| last_synced_at | LONG | NULLABLE | Epoch millis of the most recent successful sync of the operator/device row. Diagnostic only in the initial release; not currently consumed by any other component but reserves the slot for future device-level state that needs a freshness timestamp distinct from `sync_metadata.last_sync_at` (which is the route-sync freshness signal). |

**Initialisation.** A row is inserted with `id = 1`, `audio_enabled = true` (any default; will be overwritten on the first sync), and `last_synced_at = NULL` at first app launch — or lazily on the first sync. The audio downloader and playback engine treat the absence of a row as "no sync has occurred yet, behave as if audio_enabled = true" to match the pre-sync first-pairing reality (the first device paired to an operator is the audio device).

**Not synced.** This table holds only locally-derived state. It is never uploaded; the source of truth is `devices.audio_enabled` in Supabase.

### 6.1 EncryptedSharedPreferences (Secure Local Storage)

| Key | Type | Description |
|---|---|---|
| admin_pin | String | Hashed admin PIN for kiosk unlock |
| supabase_access_token | String | Current device session JWT |
| supabase_refresh_token | String | Refresh token for session renewal |
| device_secret | String | The 256-bit plaintext device secret returned by `pair-device` at pairing time. Used to authenticate `recover-device` requests. Never transmitted except to `recover-device`. |

### 6.2 Regular SharedPreferences (Local Settings)

| Key | Type | Default | Description |
|---|---|---|---|
| operator_id | String | — | Operator UUID, set during pairing |
| operator_name | String | — | Operator display name |
| device_id | String | — | This device's UUID in the devices table |
| android_id | String | — | Settings.Secure.ANDROID_ID. Passed to `recover-device` as the device row lookup key (non-secret identifier). Authentication of the recovery request relies on `device_secret` stored in EncryptedSharedPreferences, not this value. |
| screen_calibration_ppmm | Float | -1 | Pixels per mm from bank card calibration. -1 = uncalibrated. |
| auto_timeout_minutes | Int | 15 | Journey auto-timeout at final stop |
| min_volume_percent | Int | 30 | Minimum volume floor as percentage of max |
| setup_complete | Boolean | false | Whether first-run pairing has been completed |
| announcement_overlay_seconds | Int | 8 | How long announcement overlays remain visible |

### 6.3 Bundled Assets (APK Resources)

| Asset | Format | Approximate Size | Description |
|---|---|---|---|
| alert_chime.wav | WAV (LINEAR16 24 kHz mono) | ~100 KB | Alert tone before termination, diversion, and hail-and-ride announcements (~1 second; if shorter, smaller proportionally) |
| silent_keepalive.wav | WAV (LINEAR16 24 kHz mono) | ~48 KB | Inaudible 1-second audio for Bluetooth speaker keepalive |
| termination.wav | WAV (LINEAR16 24 kHz mono) | ~300 KB | Pre-rendered fixed announcement: "This service terminates here. All change, please." (~3 s at LINEAR16 mono 24 kHz ≈ 144 KB; a slightly longer rendering plus the WAV header is comfortably under 300 KB) |
| hail_and_ride_start.wav | WAV (LINEAR16 24 kHz mono) | ~400 KB | Pre-rendered fixed announcement: "You are now entering a hail and ride section. Please signal the driver if you wish to alight." (~6 s) |
| hail_and_ride_end.wav | WAV (LINEAR16 24 kHz mono) | ~250 KB | Pre-rendered fixed announcement: "You are now leaving the hail and ride section." (~4 s) |
| diversion_start.wav | WAV (LINEAR16 24 kHz mono) | ~300 KB | Pre-rendered fixed announcement: "This service is on diversion. Please check the display for affected stops." (~5 s) |
| diversion_end.wav | WAV (LINEAR16 24 kHz mono) | ~250 KB | Pre-rendered fixed announcement: "This service has resumed its normal route." (~4 s) |

The five spoken-announcement files are rendered using the same Google Cloud TTS voice
(`en-GB-Neural2-B`) and the same `audioConfig` (`audioEncoding: 'LINEAR16'`,
`sampleRateHertz: 24000`) as `audio-render-worker` (§4.6) — ensuring consistent voice quality
and identical spectral characteristics across fixed and route-specific audio, and preserving
the Reg 13(4)-verified spectrum end-to-end. They are updated by shipping a new APK version —
not by Supabase Storage sync. Specific affected stops are not named in the diversion
announcement audio; they are conveyed visually via the tube-map display (see Compliance Mapping
Matrix Reg 10(1)).

**Build-time re-rendering.** The committed binaries in `res/raw/` are produced once during
Stage 2 setup by running the same TTS render path (Google Cloud TTS REST → LINEAR16 → WAV
container) against the fixed phrases above, then committing the resulting `.wav` files. This
section specifies the bundled-asset format, content, and size envelope — not the build
process. Sizes are approximate (LINEAR16 mono 24 kHz is 48 KB/s plus the 44-byte WAV header);
the actual committed binaries will sit close to these estimates.

### 6.4 Local Audio File Storage (On-Device)

Route-specific audio files synced from Supabase Storage are stored in the app's internal file
system, separate from Room and SharedPreferences.

**Root path:** `{context.filesDir}/audio/`

**File layout (version-keyed):**
```
audio/
  {operator_id}/
    {route_id}/
      {route_version}/
        route_announcement.wav
        stop_0.wav
        stop_1.wav
        ...
        stop_N.wav
```

`{route_version}` matches the version segment in the Storage path scheme (§2.8) and is the
route's `updated_at` in epoch millis. The tablet holds exactly one version per route at any
given time — when a new version is downloaded successfully, the previous version's
directory is removed.

**Bundled assets** (termination, H&R, diversion, alert chime, keepalive) remain in APK
`res/raw/`; they are not duplicated in `filesDir`.

**Journey-start gating:** Before enabling the "Start Journey" button for a route, the app
checks **both**:
1. The Room `routes.audio_render_status` for that route is `'ok'`. If it is `pending` or
   `failed`, the gate is closed regardless of file presence.
2. All expected audio files exist in `filesDir` under the current `{route_version}` directory.
   Expected files are derived from the route's stop count (one `route_announcement.wav` +
   one `stop_{N}.wav` per stop).

If either check fails, the route shows an "Audio not ready — syncing" indicator and cannot
be started. This is a clear error state; there is no fallback to on-device TTS.

**Cleanup:**
- When a route's `is_deleted = true` is applied locally (during sync or at journey end for
  pending-deletion routes), the app deletes the corresponding `{route_id}/` directory and
  all its version subdirectories.
- When a new `{route_version}` is downloaded successfully for an existing route, older
  `{route_version}` directories for that route are removed.

**No Room schema change for audio files:** Audio file presence is determined by file-system
stat at runtime. No audio-file metadata table is needed. The `routes.audio_render_status`
column added to Room (§5.1) captures only the server-side render outcome, not local file
presence.

---

## 7. Sync Strategy

### 7.1 Sync Triggers

Route sync is initiated by three core triggers:

1. **ConnectivityManager (offline → online transition):** When Android detects the device going online, the app initiates an immediate sync. Handles the case where the tablet was offline when routes changed.
2. **FCM push notification (responsive):** When a route is inserted, updated, or deleted on the dashboard, a Supabase Edge Function sends an FCM push notification to all of that operator's active devices, prompting an immediate sync. This is the mechanism that delivers sub-5-minute route propagation to a tablet that is already online. FCM is a Must-Have feature (see §9 MoSCoW in the PRD).
3. **Periodic check (safety net):** WorkManager schedules a sync every 30 minutes if continuously online. This is a backstop for the rare case where both FCM and the connectivity trigger are missed.

Note: the heartbeat (§7.7) is a separate mechanism from route sync. It updates `devices.last_seen_at` independently and does not trigger a sync.

### 7.2 Sync Algorithm — Download

1. Read `last_server_timestamp` from sync_metadata.
2. Call the `get_routes_since(p_last_ts)` RPC. The RPC returns the route data and the server's `current_timestamp` from the transaction.
3. For each returned route:
   - If `is_deleted = true`: check whether this route is currently active in journey_state. If active, set `pending_deletion = true` locally. If not active, mark `is_deleted = true` locally (and delete its local audio folder — see §6.4).
   - Otherwise: UPSERT the route into Room.
4. For each non-deleted returned route, replace its stops in Room: delete all local route_stops for that route_id, then insert the complete set from the RPC response. This is safe because no local tables have foreign key references to route_stop IDs.
5. Update `sync_metadata.last_server_timestamp` to the `server_now` returned by the RPC.
6. Update `sync_metadata.last_sync_at` and set `sync_status = 'synced'`.
7. **Download audio files (version-keyed):** For each route UPSERT'd in step 3 (not deleted), download audio files from Supabase Storage to local file storage (§6.4) using the route's version-keyed path scheme (§2.8):
   - **audio_enabled guard (first).** Before iterating routes for audio download, read the local device's `audio_enabled` value from the `device_state` table (§5.9). **If `audio_enabled = false`, skip the entire audio-download step** — no `route_announcement.wav` or `stop_{N}.wav` is downloaded. Display-only tablets do not need route audio and do not waste storage or cellular bandwidth fetching it. Route data and `route_stops` are still downloaded normally (steps 1–6 above); only this audio-download step is skipped.
   - **Render-status guard.** If the route's `audio_render_status = 'failed'`, do **not** attempt any audio download for it — the version-keyed path will not exist in Storage and the request would 404. Log `AUDIO_FILE_MISSING` once with `detail = 'render_failed'` and move on. The "Audio not ready" indicator (§6.4) continues to gate journey starts. If `audio_render_status = 'pending'`, also defer download — a future sync (after the render worker completes) will see `ok`.
   - **For routes with `audio_render_status = 'ok'`:** compute `route_version` as the route's `updated_at` in epoch millis. The expected file list is `route_announcement.wav` plus `stop_{N}.wav` for each stop, all under `{operator_id}/{route_id}/{route_version}/`.
   - If the route was updated in this sync (it appears in the `get_routes_since` response), the path's `route_version` segment has changed; download all audio files for the new version into a fresh local `{route_id}/{route_version}/` directory (see §6.4 layout note). The previous local version directory is removed once the new download completes successfully.
   - If a file is missing locally for the current version, download it.
   - The tablet pulls audio for the specific route version it just synced; older or newer Storage versions are not pulled. If the dashboard saves the route again while the tablet is mid-download, the tablet completes its download of the version it asked for; the next sync round picks up the newer version.
   - Audio download failures (network drop, 404 on a partially-rendered version) are non-fatal: log `AUDIO_FILE_MISSING` to journey_events, continue sync. The route shows "Audio not ready" until the next successful download.
8. Update the device's `last_seen_at` in Supabase (separate query). **Detect `audio_enabled` flip from `false` → `true`.** During this step the tablet also re-reads its own `devices` row and compares the freshly-synced `audio_enabled` against the value held in `device_state.audio_enabled` (§5.9) immediately before this sync. If it has flipped from `false` to `true`, set `sync_metadata.last_server_timestamp = 0` so that the **next** sync trigger re-pulls every current route — at which point the (now `true`) `audio_enabled` guard in step 7 above lets the audio-download step run and back-fills audio for every route the tablet holds. The flip-detection happens here; the back-fill happens on the next sync, keeping the current sync's semantics simple. The reverse flip (`true → false`) does not delete already-downloaded audio in the initial release — it is dead but harmless on disk and is reclaimed on the next route delete or app reinstall. After flip-detection runs, update `device_state` with the newly-synced `audio_enabled` value and `last_synced_at = now()` (epoch millis). This write is the final action of the sync, so the cached value in `device_state` is always coherent with the most recent successful sync; any failure during the sync leaves the previous `device_state` row in place to be compared against on the next attempt.

### 7.4 Sync Algorithm — Full Sequence

1. Set `sync_metadata.sync_status = 'syncing'`.
2. Check operator account status by querying the `operators` row. Read `journey_state.is_active` from local Room.
   - **If `journey_state.is_active = true`:** record the returned status in `sync_metadata.pending_account_status` (a new local column, see note below) and **continue sync normally**. The UI is **not** locked, regardless of whether the status is `pending`, `active`, or `suspended`. Acting on a non-active status mid-journey would tear down the passenger display on a bus carrying passengers in service. The captured status is honoured at journey end (see §7.8 and PRD FR-AT-60).
   - **If `journey_state.is_active = false`:** apply the status check as a hard gate:
     - `status = 'pending'`: display "Account pending approval — contact your administrator" and abort.
     - `status = 'suspended'`: display "Account Suspended — please contact your bus company administrator" and abort.
     - `status = 'active'`: clear any previously captured `pending_account_status` and continue.
3. **Download** remote route and stop changes per section 7.2 steps 1–6. The route rows pulled by `get_routes_since` carry the new `audio_render_status` and `audio_render_error` columns (§2.4); these are written into the local Room `routes` mirror (§5.1) and used by the audio-download step below.
4. **Download audio files** per section 7.2 step 7, using version-keyed paths derived from each route's `updated_at` and skipping any route whose `audio_render_status` is `failed` or `pending`. Audio download runs after route data is committed to Room. Audio failures do not fail the sync — routes are updated even if audio files are temporarily unavailable. **Display-only tablets (`audio_enabled = false`) skip this step entirely** per the §7.2 step 7 `audio_enabled` guard; route data is still synced normally.
5. **Upload pending journey summaries** per §7.8. Failures here are non-fatal — rows remain in `journey_summaries_pending` for the next sync.
6. Set `sync_metadata.sync_status = 'synced'`.
7. Update `devices.active_route_id` if a journey is in progress.
8. On any failure in steps 1–3 or 6, set `sync_status = 'failed'` and retry on next trigger. Audio download failure (step 4) and journey-summary upload failure (step 5) are logged but do not set `sync_status = 'failed'`.

**Mid-journey suspension grace.** Mid-journey suspension is honoured at journey end, not mid-journey. The cost — one final journey of "unpaid" service after suspension — is acceptable and recoverable via invoicing. The humane-and-operationally-correct behaviour is to never tear down a passenger display while passengers are aboard.

**Note on `sync_metadata.pending_account_status`.** This column is added to §5.6 implicitly as a single nullable TEXT field on the single-row `sync_metadata` table. Values: NULL (no pending non-active status captured), `'pending'`, or `'suspended'`. Cleared when a subsequent sync sees `status = 'active'` (step 2 above) or when the Account Status Screen is dismissed after a return to active (FR-AT-60). The journey-end transition (§7.8) reads this column to decide whether to route to the route list or to the Account Status Screen.

### 7.5 Conflict Resolution

In the initial release, conflicts cannot occur because tablets are read-only for routes. Only the dashboard writes routes, and only one dashboard user exists per operator (FR-WD-06).

### 7.6 FCM Push Notifications

**Dispatch point.** Route-change FCM push is fired by the **`audio-render-worker`** on
successful job completion (§4.6 per-job step 10), **not** by a routes-table trigger and
**not** by the dashboard at save time. This is the render-then-FCM ordering: a push fires
only after the route's audio is available in Storage, guaranteeing that a tablet which syncs
in response to the push will find both the route data and the version-keyed audio ready to
download. A render that ends in terminal failure does **not** fire FCM; the failed status
reaches tablets only on their next regular sync trigger.

**Dispatcher.** The dispatcher itself is a small server-side function (callable by the worker
via service-role HTTP) that:
1. Takes `(route_id, trigger)` as input, where `trigger` is `'route-updated'` or `'route-deleted'`.
2. Queries `routes` to resolve `operator_id`.
3. Queries `devices` for all devices belonging to that operator with `activation_state = 'active'` AND `fcm_token IS NOT NULL`. Inactive devices and devices that have not yet registered an FCM token are skipped.
4. Sends an FCM **data-only message** (see payload spec below) to each device's FCM token.

The dispatcher shape is unchanged from earlier versions in its mechanics; the payload shape
and the dispatch filter (`activation_state = 'active'`) are made explicit here.

**Payload (data-only message).** FCM messages are sent as Android **data messages**, not
notification messages. A data-only message wakes the app to handle the payload in code; a
notification-with-data message would render a system tray notification when the app is
backgrounded, which is wrong for a kiosked always-foregrounded application. Payload shape:

```json
{
  "data": {
    "type": "route-sync",
    "operator_id": "<uuid>",
    "trigger": "route-updated"
  }
}
```

- `type` — fixed string `"route-sync"`. Future message classes (if ever added) would use a
  different value; the Android handler dispatches on `type` and ignores unknown values
  defensively.
- `operator_id` — the owning operator's UUID. The Android FCM handler validates this against
  the locally-stored `operator_id` and ignores any message whose `operator_id` does not
  match. In practice each device's FCM token is operator-specific by registration, but the
  explicit check is defence-in-depth against misrouted messages and costs nothing.
- `trigger` — informational only; one of `"route-updated"` or `"route-deleted"`. Useful for
  Sentry breadcrumbs and operational diagnostics. The handler does not branch on this
  value — every `route-sync` message produces the same response (trigger an immediate sync).

Tablets register their FCM registration token in `devices.fcm_token` (see §2.2) during the
first successful sync after pairing. The token is re-registered whenever Firebase rotates it.
On receiving an FCM message, the tablet triggers an immediate sync.

FCM is the mechanism that enables sub-5-minute route propagation **from successful render**
(PRD §12 success metric — note that the metric now starts the clock at render completion,
not at dashboard save, because FCM no longer fires at save). If FCM is unavailable (device
offline, FCM service down), the connectivity-change and 30-minute periodic triggers ensure
eventual consistency — propagation may take up to 30 minutes in the worst case.

### 7.7 Heartbeat Mechanism

The heartbeat is a lightweight periodic update that keeps `devices.last_seen_at` current whenever the tablet has connectivity. It is independent of route sync: it does not fetch routes, process changes, or update any other state. It runs whenever the app is in the foreground (whether or not a journey is active) and best-effort when the app is backgrounded.

**Purpose.** The dashboard's fleet view uses `last_seen_at` to show online/offline status. Without a heartbeat, `last_seen_at` only updates when a route sync occurs (every 30 minutes at most), causing healthy in-service tablets to appear offline. The heartbeat ensures `last_seen_at` is accurate throughout the device's operating day — including when the driver has the tablet on but no journey is active (e.g., browsing the route list, viewing route detail, working in the admin menu). This "foregrounded-but-idle" state was previously uncovered by the heartbeat and is the case this redesign addresses.

**Interval.** 2 minutes.

**Online threshold.** A device is considered online if `last_seen_at` is within the last 5 minutes. The 2-minute heartbeat interval plus a 3-minute margin for network hiccups gives a 5-minute threshold that a healthy tablet reliably meets.

**Ownership.** Heartbeats are owned by an **application-level lifecycle observer** — a Hilt-injected singleton (e.g., `HeartbeatController`) that observes `ProcessLifecycleOwner.get().lifecycle`. The foreground GPS service is **not** the heartbeat owner. The GPS service exists for stop detection during journeys; the heartbeat lives at a higher level so that its lifecycle is tied to the app being foregrounded, not to whether a journey is in progress. Decoupling these two responsibilities closes the previous third-state gap (foregrounded but no journey active) and makes each component single-purpose.

**Implementation — two-path:**

1. **App-foregrounded path (reliable).** A `Handler`-based ticker tied to `ProcessLifecycleOwner.get().lifecycle`:
   - On `ON_RESUME` (any Activity reaches the `RESUMED` state), `HeartbeatController` starts a `Handler.postDelayed` loop with a 2-minute interval. The first tick fires immediately so the app announces "I'm here" on resume.
   - On `ON_PAUSE` (the last `RESUMED` Activity leaves `RESUMED`), the ticker is cancelled.
   - This path covers every state in which the driver is actively using the app: route list, route detail, admin menu, active journey, settings, etc. Because the app is foregrounded, the OS does not throttle this loop — it is as reliable as the app itself is.

2. **Background/idle path (best-effort).** A WorkManager `PeriodicWorkRequest` with a 2-minute interval runs only when the app is backgrounded or the screen is off:
   - The controller observes `ON_STOP` and schedules the periodic work; on `ON_START` it cancels the periodic work (the foreground ticker has resumed).
   - WorkManager may be delayed or throttled by OEM scheduling on aggressive hardware. This is the OEM best-effort caveat that has always applied to the idle path.

**Database operation.** A single `UPDATE devices SET last_seen_at = now() WHERE id = :device_id` query using the device's existing Supabase session. No payload, no route data.

**Failure handling.** Heartbeat failures are silent. No UI indication, no retry. The next successful heartbeat updates the timestamp. A failed heartbeat does not affect journey operation.

**OEM best-effort caveat (background/idle path).** The WorkManager `PeriodicWorkRequest` used for the background/idle heartbeat is best-effort on hostile-OEM cheap tablets — the same OEM category flagged in the PRD risk table for foreground service killing. Aggressive OEM battery management can throttle or delay WorkManager tasks in ways that `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` does not address. As a result: **the foregrounded path (Handler ticker on ProcessLifecycleOwner) is reliable**; the **background/idle WorkManager path is not guaranteed on hostile hardware**. A tablet that is powered off, fully backgrounded, in aggressive doze mode, or managed by a hostile OEM may show offline in the fleet view until the app is foregrounded again. This is acceptable and documented behaviour — the critical online-status window is during journeys and immediately around journey start/end, which the foregrounded path reliably covers. Fleet managers should interpret idle (un-foregrounded) tablets as "not in active use" rather than "connectivity failure."

**Relationship to GPS service.** The foreground GPS service is started at journey start and stopped at journey end. Its only job is GPS-based stop detection. It does not own, fire, or supervise the heartbeat. When a journey starts, the heartbeat continues running on its existing foreground-path ticker (the journey-start activity is `RESUMED`); when a journey ends, the ticker continues until the activity is paused or the app is backgrounded.

**Relationship to sync.** The heartbeat does not replace or interact with the three sync triggers (§7.1). Sync remains triggered by ConnectivityManager, FCM, and the 30-minute WorkManager periodic task. The heartbeat only updates `last_seen_at`.

### 7.8 Journey-Summary Upload

At journey end, the tablet writes a single anonymous-count summary row to local Room (§5.8 `journey_summaries_pending`) and uploads queued summaries during sync into the Supabase `journey_summaries` table (§2.6). This is **not a real-time stream and does not violate offline-first** — it's a small post-journey payload sent during the post-journey sync that already happens.

**At journey end (write to local pending table):**

1. Triggered by any journey termination — driver-ended via the End Journey action, journey auto-timeout, or natural arrival at the final stop.
2. **Before clearing `journey_state`**, build a `journey_summaries_pending` row by aggregating the just-completed journey's `journey_events` rows:
   - `stops_announced_count = COUNT(*) WHERE event_type = 'STOP_ANNOUNCED'`
   - `stops_passed_without_detection_count = COUNT(*) WHERE event_type = 'STOP_PASSED_WITHOUT_DETECTION'`
   - `manual_advances_count = COUNT(*) WHERE event_type = 'STOP_ANNOUNCED' AND trigger_method = 'MANUAL'`
   - `gps_lost_events_count = COUNT(*) WHERE event_type = 'GPS_LOST'`
   - `audio_failures_count = COUNT(*) WHERE event_type IN ('AUDIO_FILE_MISSING', 'AUDIO_PLAYBACK_ERROR')`
   - `diversion_invoked = journey_state.diversion_invoked_at_any_point` evaluated at journey end. The boolean latch on `journey_state` (§5.5) survives a mid-journey "Diversion end" that clears `journey_skipped_stops`, so the metric is correct in **all four cases**: no diversion this journey (`false`), a diversion still active at journey end (`true`), a diversion started and then ended mid-journey (`true`), and a diversion that was replayed on app restart (`true`). Reading the latch is a single-row Room lookup, cheaper and clearer than counting rows in `journey_skipped_stops`.
3. The row scoping window for `journey_events` aggregation is "events with `timestamp_utc >= journey_state.journey_started_at`" — every event since the journey began, including those logged in the same transaction as the journey-end event.
4. Insert into `journey_summaries_pending` with `needs_upload = 1`. Generate `id` as a fresh UUID; this is the same UUID that will be used as the Supabase PK on upload, making the upload idempotent on retry.
5. After the pending row is written, clear `journey_state.is_active = false` (and the rest of journey teardown follows: `journey_skipped_stops` cleared, etc.).
6. After teardown, the controller reads `sync_metadata.pending_account_status`. If non-NULL (`'pending'` or `'suspended'`), transition the UI to the Account Status Screen (FR-AT-60). Otherwise return to the route list normally.

**Setting the diversion latch.** The `journey_state.diversion_invoked_at_any_point` column (§5.5) is set to `true` by the same code path that handles the driver's diversion-start action (FR-AT-25), in the same Room transaction that writes the first `journey_skipped_stops` rows for the diversion. It is never reset while the journey remains active. On a new journey start, the entire `journey_state` row is replaced (single-row REPLACE pattern), which naturally clears the latch back to its DEFAULT `false` for the new journey.

**Upload step (during sync):**

1. As part of §7.4 step 5 (after the route data has been committed to Room and audio has been downloaded), the tablet queries `SELECT * FROM journey_summaries_pending WHERE needs_upload = 1 ORDER BY journey_started_at ASC`.
2. For each pending row, perform an INSERT into Supabase `journey_summaries` using the device's authenticated session. The device JWT scopes the insert to its own `(device_id, operator_id)` via RLS (§2.6).
3. On success (200 / 201), **delete** the row from `journey_summaries_pending`. (Deletion rather than `UPDATE needs_upload = 0` keeps the local table small and avoids accumulating historical local rows.)
4. On failure (network, RLS rejection, transient 5xx), leave the row in place with `needs_upload = 1` and set `last_upload_attempt_at = now()`. Move on to the next row; do not abort the sync.
5. Upload failures do not set `sync_status = 'failed'` — journey-summary upload is a non-critical side channel. Pending rows are simply retried on the next sync.

**Offline-first compatibility.** Because `journey_summaries_pending` is local and rows persist across restarts, a tablet can run journeys offline for days and back-fill summaries on next sync. The pending table is a short-lived queue, not a permanent record — once Supabase has the row, the local copy is deleted.

**Privacy.** Summaries contain no operator names, no driver names, no passenger information, no GPS traces, and no stop names. They are pure aggregate counts plus journey start/end timestamps. See §2.6 privacy posture.

---

## 8. Data Flow Diagrams

### 8.1 Operator Signup Flow

    User                          Dashboard                    Supabase
      |                              |                            |
      |-- Sign up form ------------->|                            |
      |   (email, password, company) |                            |
      |                              |-- supabase.auth.signUp --->|
      |                              |                            |-- Create auth.users row
      |                              |                            |-- Trigger: insert into operators
      |                              |                            |              (status='pending')
      |                              |                            |-- Trigger: send email to admin
      |                              |<-- Session JWT ------------|
      |<-- "Pending approval" view --|                            |
      |                              |                            |
      |             [System admin manually approves]              |
      |                              |                            |-- UPDATE operators SET status='active'
      |                              |                            |
      |-- Refresh page ------------->|                            |
      |                              |-- Query operators (RLS) -->|
      |                              |<-- status='active'---------|
      |<-- Full dashboard --------- |                            |

### 8.2 Device Pairing Flow

    Fleet Mgr        Dashboard        Edge Function          Supabase            Tablet
       |                |                  |                     |                  |
       |-- "Add Device" |                  |                     |                  |
       |--------------->|                  |                     |                  |
       |                |-- generate-pairing-code --------------->|                  |
       |                |                  |-- Insert pairing_code row              |
       |                |                  |-- Return code                          |
       |                |<-- code 472901 --|                     |                  |
       |<-- Show code --|                  |                     |                  |
       |                |                  |                     |                  |
       |   [Fleet manager enters code on tablet]                                    |
       |                |                  |                     |   Setup screen   |
       |                |                  |                     |<-- "472901" -----|
       |                |                  |<-- pair-device(code, android_id)-------|
       |                |                  |-- Validate code                        |
       |                |                  |-- Create anon auth user                |
       |                |                  |-- Insert devices row                   |
       |                |                  |-- Mark code used                       |
       |                |                  |-- Issue session JWT                    |
       |                |                  |-- Return JWT + device info ----------->|
       |                |                  |                     |                  |
       |                |                  |                     |   [Store JWT]    |
       |                |                  |                     |   [Trigger initial sync]

### 8.3 Active Journey Data Flow

    GPS Service          Room DB           UI (Passenger + Driver)
       |                   |                       |
       |-- Position fix -->|                       |
       |   (+ accuracy     |                       |
       |    estimate, m)   |                       |
       |                   |-- Read stop N: proximity_radius_meters = R_N
       |                   |-- Read stop N+1: proximity_radius_meters = R_N1
       |                   |                       |
       |   [Fix accuracy > R_N → fix discarded, no action]
       |                   |                       |
       |   [Fix accuracy ≤ R_N AND distance to stop N ≤ R_N]
       |                   |-- Update journey_state.current_stop_index → N+1
       |                   |-- INSERT journey_event (STOP_ANNOUNCED, trigger=GPS)
       |                   |--------- Emit StateFlow update ---------->|
       |                   |                       |-- Announce stop N (audio + visual)
       |                   |                       |-- Update tube-map: N complete
       |                   |                       |
       |   [Fix accuracy ≤ R_N1 AND distance to stop N+1 ≤ R_N1, stop N not yet registered]
       |                   |-- INSERT journey_event (STOP_PASSED_WITHOUT_DETECTION, stop N, trigger=GPS_INFERRED)
       |                   |-- INSERT journey_event (STOP_ANNOUNCED, stop N+1, trigger=GPS_INFERRED)
       |                   |-- Update journey_state.current_stop_index → N+2
       |                   |--------- Emit StateFlow update ---------->|
       |                   |                       |-- Announce stop N+1 (audio + visual)
       |                   |                       |-- Update tube-map: N passed (dimmed), N+1 complete
       |                   |
       |   [Stop N has segment_type='hail_and_ride' AND preceding stop was 'scheduled']
       |                   |-- (Segment boundary: entering H&R section)
       |                   |-- INSERT journey_event (HAIL_AND_RIDE_SECTION_STARTED, trigger=GPS)
       |                   |-- (No STOP_ANNOUNCED — H&R stops are traversed silently)
       |                   |-- Update journey_state.current_stop_index → N+1
       |                   |--------- Emit StateFlow update ---------->|
       |                   |                       |-- Play alert_chime.wav then hail_and_ride_start.wav
       |                   |                       |-- Update tube-map: H&R section rendered dashed
       |                   |
       |   [Stop N has segment_type='scheduled' AND preceding stop was 'hail_and_ride']
       |                   |-- (Segment boundary: exiting H&R section)
       |                   |-- INSERT journey_event (HAIL_AND_RIDE_SECTION_ENDED, trigger=GPS)
       |                   |-- INSERT journey_event (STOP_ANNOUNCED, stop N, trigger=GPS)
       |                   |-- Update journey_state.current_stop_index → N+1
       |                   |--------- Emit StateFlow update ---------->|
       |                   |                       |-- Play alert_chime.wav then hail_and_ride_end.wav
       |                   |                       |-- Play stop_{N}.wav
       |                   |                       |-- Update tube-map: H&R section complete
       |                   |
       |   [current_stop_index N is present in journey_skipped_stops]
       |                   |-- INSERT journey_event (STOP_SKIPPED, stop N, trigger=DRIVER_DIVERSION)
       |                   |-- Update journey_state.current_stop_index → N+1
       |                   |   (repeat immediately if N+1 is also in journey_skipped_stops)
       |                   |--------- Emit StateFlow update ---------->|
       |                   |                       |-- Update tube-map: N shown with strikethrough

### 8.4 Route Sync Flow (render-then-FCM)

    Dashboard      Supabase / Storage      pg_boss        audio-render-worker        FCM            Tablet
       |                  |                  |              (scheduled, 1 min)        |               |
       |                  |                  |                                        |               |
       |-- Save route --->|                  |                                        |               |
       |                  |-- replace_route_with_stops                                |               |
       |                  |   routes.updated_at stamped (= route_version)             |               |
       |                  |   audio_render_status = 'pending'                         |               |
       |                  |                                                           |               |
       |-- enqueue-render-job(route_id, route_version) -->|                           |               |
       |                  |                  |-- boss.send                            |               |
       |<-- {job_id} -----|                  |                                        |               |
       |                  |                  |                                        |               |
       |                  |                  |   [every minute]                       |               |
       |                  |                  |<--------- boss.fetch ------------------|               |
       |                  |                  |---------- job batch ------------------>|               |
       |                  |                                                           |               |
       |                  |                              [Per-job processing:]                        |
       |                  |                              Staleness check: routes.updated_at == route_version?
       |                  |                              Compute would-be hashes for announcement + each stop
       |                  |                              For each: hash matches stored? && file at target path?
       |                  |                                  yes → skip (no-op)
       |                  |                                  hash matches but no file at new version path →
       |                  |<-- Storage server-side copy from prev version ------------|               |
       |                  |                                  hash differs or no prior version →
       |                  |                              Google TTS synthesize (en-GB-Neural2-B) ----> Google Cloud
       |                  |<-- Storage PUT at {operator_id}/{route_id}/{route_version}/...wav --------|
       |                  |    UPDATE route_stops.audio_content_hash                  |               |
       |                  |    UPDATE routes.audio_announcement_hash                  |               |
       |                  |                  |                                                        |
       |                  |                  |-- boss.complete (per job)                              |
       |                  |    UPDATE routes.audio_render_status = 'ok'                               |
       |                  |                  |                                                        |
       |                  |                                                            FCM dispatch -->|
       |                  |                                                                           |
       |                  |                                                                  [Sync triggered]
       |                  |<-- get_routes_since(last_ts) -------------------------------------------- |
       |                  |--- routes (incl. audio_render_status='ok', route_version=updated_at) --->|
       |                                                                                             |-- UPSERT routes (incl. status)
       |                                                                                             |-- Replace route_stops
       |                                                                                             |-- Update last_server_timestamp
       |                  |                                                                          |
       |                  |<-- Storage GET {operator_id}/{route_id}/{route_version}/*.wav -----------|
       |                  |--- Stream WAV files (LINEAR16 PCM) ----------------------------------->  |
       |                                                                                             |-- Store in filesDir/audio/{operator_id}/{route_id}/{route_version}/

    Failure branch (terminal, after pg_boss retries exhausted):
       |                  |   UPDATE routes SET audio_render_status='failed', audio_render_error=$msg
       |                  |   (FCM is NOT fired)
       |                  |   Failure surfaces on dashboard route list (FR-WD-13);
       |                  |   "Re-render audio" action (FR-WD-21) re-enqueues.
       |                  |   Tablets see the 'failed' status on their next regular sync and skip audio download.

### 8.5 Heartbeat Flow

    App (foregrounded — any Activity RESUMED)        Supabase
    HeartbeatController (Hilt singleton,                |
      ProcessLifecycleOwner observer)                   |
       |                                                |
       |-- [on ON_RESUME, immediately] ---------------->|
       |   UPDATE devices SET last_seen_at = now()      |
       |<-- 200 OK -------------------------------------|
       |                                                |
       |-- [every 2 minutes, Handler.postDelayed] ----->|
       |   UPDATE devices SET last_seen_at = now()      |
       |<-- 200 OK -------------------------------------|
       |                                                |
       |   [on ON_PAUSE: ticker cancelled,              |
       |    WorkManager periodic work scheduled]        |
       |                                                |
       |   [on failure: log silently, no retry]         |

    WorkManager (app backgrounded / screen off)      Supabase
       |                                                |
       |-- [every 2 minutes, PeriodicWorkRequest] ----->|
       |   UPDATE devices SET last_seen_at = now()      |
       |<-- 200 OK -------------------------------------|
       |                                                |
       |   [on ON_START: periodic work cancelled,       |
       |    foreground ticker resumed]                  |
       |                                                |
       |   [on failure: log silently, no retry]         |
       |   [next scheduled run retries naturally]       |
       |   [OEM best-effort caveat on hostile hardware] |

    Note: the foreground GPS service is NOT involved in the heartbeat.
    Heartbeat lifecycle is owned by HeartbeatController at the app level,
    not by the GPS service. Journey activity does not affect heartbeating
    other than via the app remaining foregrounded during a journey.

---

## 9. Entity Relationship Summary

    auth.users (Supabase Auth)
        |
        +-- (1:1) --> operators                          (one dashboard user per operator)
        |                  |
        |                  +-- (1:many) --> devices       (operator owns devices)
        |                  |                    |
        |                  |                    +-- (1:1) --> auth.users  (each device is also an auth user)
        |                  |
        |                  +-- (1:many) --> routes
        |                  |                    |
        |                  |                    +-- (1:many) --> route_stops
        |                  |
        |                  +-- (1:many) --> journey_summaries   (operator-scoped, anonymous counts)
        |
        +-- (1:1) --> devices                              (alternative path: tablet user)

    device_pairing_codes: standalone, short-lived. Linked to operators by FK.
    naptan_stations:      standalone reference data, shared across all operators.

    Local (Room) tables: routes, route_stops,
                         journey_events, journey_state, sync_metadata,
                         journey_skipped_stops, journey_summaries_pending

---

## 10. Data Retention and Cleanup

| Data | Retention | Cleanup |
|---|---|---|
| Journey events (local) | 30 days | Auto-deleted on app startup |
| Used/expired pairing codes | 1 hour | Scheduled cleanup function in Supabase |
| Soft-deleted routes | Indefinite in Supabase | **No scheduled hard delete in the initial release.** Soft-deleted routes are retained indefinitely so that `get_routes_since` always propagates the deletion to every tablet that hasn't yet seen it. Hard deletion would silently strand routes on tablets that synced before the delete (`get_routes_since` cannot return rows that no longer exist) — tombstone-table alternatives are over-engineering for the data volumes involved. A manual administrative hard-delete remains possible (and the schema's `ON DELETE SET NULL` / `ON DELETE CASCADE` rules cover it cleanly) but is not part of automated retention. |
| Journey state (local) | Cleared on journey end **or on stale-recovery auto-clear (PRD FR-AT-18)** | `is_active = false` when journey completes; pending_deletion cleanup runs here too. On app launch, journey state older than 8 hours or with no `journey_events` activity in the last hour is auto-cleared (see §5.5 staleness rules; logs `JOURNEY_AUTO_CLEARED`). |
| Active vs auto-deregistered devices | **30 days heartbeat-billable; 60 days auto-deregister (compound)** | A scheduled daily job (sharing the 03:00 UTC cleanup window) sets `devices.activation_state = 'inactive'` for any device that satisfies **both**: (a) `last_seen_at` older than `now() - interval '60 days'`, AND (b) `recover_failure_count >= 5` with `last_recover_failure_at` within the trailing 60 days (i.e. `last_recover_failure_at > now() - interval '60 days'`). The compound condition distinguishes two qualitatively different cases that the silence-alone rule conflated: a tablet being **actively rejected by the backend** (operator decommissioned the device, secret mismatch, hard-deleted row) is producing terminal `recover-device` failures as it tries to come back online — the failure counter accumulates and auto-deregistration is correct; a tablet that is **simply silent** (seasonal route paused, in-repair, OEM-throttled background, powered off) is not attempting recovery at all — its failure counter stays at zero and it is **never** auto-deregistered. The silent case retains its `devices` row indefinitely so the operator can power the tablet back on whenever the season returns and have it recover seamlessly, without forcing a re-pair. Billing reports continue to exclude any device whose `last_seen_at` is older than `now() - interval '30 days'`, regardless of `activation_state` — so a silent seasonal tablet stops being billed but is not auto-deregistered. **`last_seen_at` is still the source of truth for the silence half**, maintained by the heartbeat mechanism (§7.7 / PRD FR-AT-64) rather than by route sync alone. `recover_failure_count` is incremented inside the `recover-device` terminal-failure path (§4.3 failure classification) and reset to 0 on every successful `recover-device`; `last_recover_failure_at` is overwritten on every failure. The 30-day grace gives operators a buffer for billing; the 60-day-plus-failures gate bounds the lifetime of rows that the backend has positively confirmed are no longer in legitimate operation. This pairing reconciles PRD §1.4 (billing) with this retention policy. Resolves round-3 finding 13 (the seasonal-tablet false-positive deregistration case). |
| Anonymous Supabase Auth users | Indefinite (known unbounded growth) | Every pair-device creates a new anonymous `auth.users` row; device deactivation flips `devices.activation_state = 'inactive'` but does not remove the auth user. MVP-acceptable at projected fleet size; a future cleanup task (delete `auth.users` rows for `inactive` devices with no activity in the last 12 months) is recorded as out-of-scope but acknowledged — see §3.2 paragraph. |
| Route audio (Storage) | Three most-recent versions per route | Daily `audio-cleanup-worker` (§4.7) at 03:00 UTC; older `{route_version}` paths removed. (Increased from two in v3.9 — see §4.7 rationale and the named rapid-iteration assumption.) |
| pg_boss jobs | 7 days after completion or terminal failure | `retentionDays: 7` on the `render-route-audio` queue (§4.6); pg_boss archives to `pgboss.archive` then prunes |
| Journey summaries (Supabase) | Indefinite | Operator-visible in the dashboard per-device drill-down (PRD FR-WD-23). No automatic deletion; the system administrator may prune via service role if storage cost ever becomes a concern. No PII to protect. |
| Journey summaries pending (local) | Until successful upload | Deleted from `journey_summaries_pending` (§5.8) on a successful INSERT into Supabase `journey_summaries`. Rows survive app restarts; the upload step retries on every sync. |

---

## 11. Implementation Notes

### 11.1 Cross-Cutting Implementation Notes

A few non-obvious points worth capturing here so they're not lost:

**UUIDs as TEXT in Room.** Supabase UUIDs are stored as TEXT strings in Room. No type conversion at the boundary. Simpler than the alternatives.

**Single-row Room tables.** journey_state and sync_metadata are single-row tables. Implementation pattern: `@PrimaryKey(autoGenerate = false)` with id hardcoded to 1, and `@Insert(onConflict = OnConflictStrategy.REPLACE)` in the DAO. This prevents `SQLiteConstraintException` from race conditions where two insert paths run concurrently — any insert overwrites the existing row.

**Stops never sync independently.** Route stops do not have their own `updated_at` or `is_deleted`. They are part of their parent route and travel with it. The `replace_route_with_stops` RPC handles atomic replacement.

**Route stops copy NaPTAN data.** When a route is created on the dashboard, the stop's name, CRS code, latitude, and longitude are copied into the route_stops row. The route is not a foreign-key reference to naptan_stations. This means routes survive NaPTAN database changes (a station rename in NaPTAN does not change saved routes), and tablets don't need a current NaPTAN snapshot to use synced routes.

**pending_deletion handling.** When a remote sync indicates a route has been deleted, and that route happens to be the currently active journey, we set `pending_deletion = true` rather than `is_deleted = true`. The journey continues uninterrupted on the cached version. When the journey ends, cleanup converts pending_deletion to is_deleted.

**Sync cursor uses server transaction time, not max(updated_at).** The `get_routes_since` RPC returns `current_timestamp` from its transaction as the cursor, not the maximum `updated_at` from its result set. This prevents the device re-downloading rows that were updated during the read.

**Audio file presence gating.** Before enabling the "Start Journey" button for a route, the app
checks two things: (1) the Room `routes.audio_render_status` is `'ok'`, and (2) `File.exists()`
on each expected audio file under the current `{route_version}` directory. If either check
fails, the route renders with an "Audio not ready — syncing" (or "Audio render failed" if
status is `failed`) indicator and the button is disabled. There is no fallback to on-device
TTS. On-device TTS is not used anywhere in this product.

**Differential re-rendering via content hashes.** Each route's `routes.audio_announcement_hash`
and each stop's `route_stops.audio_content_hash` capture the SHA-256 of the text most recently
rendered to the corresponding WAV file. On re-render, the audio render worker (§4.6) computes
the would-be hashes for the current text, compares against the stored hashes, and only
renders stops whose hash has changed. Unchanged audio is server-side-copied from the previous
version's Storage path into the new version's path, so a direction-label-only edit (which
changes no announcement text) calls Google TTS zero times.

**Version-keyed Storage paths.** Each route save produces a new `{route_version}` path prefix
(`routes.updated_at` in epoch millis). Tablets pull audio for the exact route version they
synced; older versions remain in Storage for a recovery window of three versions (cleanup
policy in §4.7). This makes concurrent saves race-safe by construction — two saves cannot
collide on the same path.

**Voice is locked.** Google Cloud TTS voice `en-GB-Neural2-B` is fixed in code; it is not a
runtime configurable. Changing the voice is a deliberate compliance event — Reg 13(4)
frequency verification (Compliance Mapping Matrix) must be re-run before any new voice ships.
Treat the locked voice as an inviolable architectural rule alongside the alert-chime
sequence.

**pg_boss owns its schema.** The `pgboss` schema is managed entirely by the pg_boss library;
do not write application tables in it, do not migrate it manually, and do not couple
application code to specific pg_boss internal table shapes — pg_boss handles its own schema
versioning across library upgrades.

**Audio download is non-fatal.** A failed audio file download does not set `sync_status =
'failed'` and does not prevent the route data from being written to Room. The route is
usable once audio files are present and `audio_render_status = 'ok'`; the "Audio not ready"
indicator disappears after the next successful download.

### 11.2 Third-Party Service Dependencies

**Sentry (sentry.io)** — the crash-reporting and error-tracking SDK for all three surfaces:
the Android tablet app, the Next.js dashboard, and Supabase Edge Functions. Sentry's free
tier (5,000 errors/month) is expected to be sufficient at projected fleet size; the system
administrator monitors quota usage and can upgrade if/when needed.

**Quota budget (v3.9).** Concrete expected steady-state volume at the 20-tablet pilot scale,
sized to validate that the free tier is comfortably sufficient rather than asserted on
faith:
- **Android tablet crashes / ANRs:** ~1 per tablet per week × 20 tablets × ~4.3 weeks/month ≈ **80 events/month**. Cheap-OEM tablets at the bottom of the approved-hardware list are the dominant source.
- **Dashboard (Next.js) errors:** typically **<10/month** at small-operator scale. Single-user-per-operator dashboards produce errors at very low rates dominated by genuine bugs surfaced during development and rare network blips during route saves.
- **Edge Function errors** across `pair-device`, `recover-device`, `generate-pairing-code`, `enqueue-render-job`, `audio-render-worker`, `audio-cleanup-worker`, and `retry-admin-notification`: typically **<50/month combined**. The largest expected single source is transient Google Cloud TTS API errors from `audio-render-worker`, which pg_boss retries absorb but Sentry reports for visibility.
- **Total steady state:** ≈ **140 events/month** across all three surfaces.
- **Headroom:** 5,000 − 140 = **≈ 4,860 events/month** (≈ 35× the steady-state rate). Comfortably absorbs unusual events such as a bad firmware push that crashes every tablet on the same release, or a sustained Google TTS API outage that produces hundreds of Edge Function errors before the operator notices.

**Counts toward quota.** Only **errors and crashes** count. **Transactions are disabled** in all three Sentry SDKs (`tracesSampleRate: 0`) to preserve the budget — performance tracing is not used in this product. Breadcrumbs are unlimited and do not count against the quota; they attach to events that do count.

**Quota-warning response.** Sentry sends quota-warning emails at 80% and 100% of the monthly ceiling. On warning, the system administrator: (1) upgrades to a paid tier if the trigger reflects sustained higher volume from real growth (paid tier ≈ $26/month for the smallest plan, gives 50k events/month); or (2) temporarily lowers sample rates and/or silences the noisiest fingerprint via the Sentry UI if the trigger is a one-off bug producing redundant events that have already been root-caused. Neither response is automated.

**Project layout.** Three separate Sentry projects, one per surface, each with its own DSN.
DSN values are delivered via environment variables — see §12. Three projects (rather than
one shared project) keeps error volumes, release tags, and alerting rules cleanly separated
by surface.

**Android (tablet).** The Sentry Android SDK is initialised at `Application.onCreate()`. It
captures:
- Unhandled exceptions (including those in coroutines and background workers when configured).
- ANRs (Application Not Responding) on the main thread.
- Session crash counts for release-health metrics.
Breadcrumbs include journey-lifecycle events (`route_id`, `current_stop_index`,
`device_id` — all diagnostic, no PII), GPS state transitions (`GPS_LOST`, `GPS_REGAINED`),
sync outcomes, and audio events. **No operator names, no passenger information** — the
system contains no end-user PII at all. **`device_id` is a UUID identifier with no personal
information embedded; it is permitted in Sentry breadcrumbs as a diagnostic correlation key.**
It is the bridge between a captured crash record and the per-device drill-down in the
dashboard fleet view (PRD FR-WD-23), and is essential for fleet-scale debugging — without
it, a crash report cannot be linked to "which tablet is this and what was its operating
context." `device_id` is not personal data and is not used for marketing, advertising, or
any non-diagnostic purpose; the per-device crash count is consumed only for fleet-health
diagnosis.

**Dashboard (Next.js).** The `@sentry/nextjs` SDK is wired in via the standard Next.js
config. It captures:
- Client-side React errors (error boundaries and unhandled promise rejections).
- Server-side rendering errors.
- Server-action errors (route creation, pairing-code generation, audio re-render action).

**Edge Functions.** The Sentry SDK for Deno (or the relevant runtime) is initialised at the
top of each Edge Function. Errors are reported for:
- `pair-device`
- `recover-device`
- `generate-pairing-code`
- `enqueue-render-job`
- `audio-render-worker`
- `audio-cleanup-worker`
- `retry-admin-notification`

**Failure mode.** The Sentry SDK fails open. If Sentry itself is unavailable or
rate-limited, application behaviour is unaffected — errors that would have been reported
are dropped silently rather than blocking the request path.

**One-off operational setup.** Sentry account/project creation and DSN provisioning is a
one-off operational task performed at Stage 2/3 start (when the Android app and dashboard
exist). It is not a per-deployment or per-release task. See the §12 setup checklist.

**Google Cloud Text-to-Speech (output format note, v3.9).** Although the primary Google
Cloud TTS contract is described in §4.6 and PRD §11.2, two output-format details bear
mention here because they affect Storage, Sync, and Reg 13(4) compliance: (1) the TTS API
call must request `audioEncoding: 'LINEAR16'` and `sampleRateHertz: 24000` (the voice's
native rate); (2) the rendered output is stored as a canonical PCM WAV file (`.wav`) — the
worker prepends a 44-byte WAV header to the raw LINEAR16 bytes returned by the API. MP3 is
**not** used anywhere in the pipeline. This is part of the Reg 13(4) compliance argument:
no lossy compression sits between the verified-spectrum renderer and the tablet's speaker
(Compliance Mapping Matrix Reg 13(4); spike record at
`spike-records/round-3/findings-tts-frequency.md`).

---

## 12. Environment Variables and Secrets

Server-side secrets are managed as **Supabase Edge Function secrets**, set via
`supabase secrets set NAME=value`. They are injected into every Edge Function's environment
at invocation time.

| Variable | Used by | Purpose |
|---|---|---|
| `SUPABASE_URL` | All Edge Functions, dashboard | Project URL (built-in for Edge Functions). |
| `SUPABASE_ANON_KEY` | Dashboard, tablet | Anonymous client key. Tablet uses this for `pair-device` and `recover-device` calls before it has a session. |
| `SUPABASE_SERVICE_ROLE_KEY` | All Edge Functions | Bypasses RLS. Required by `pair-device`, `recover-device`, `generate-pairing-code`, `enqueue-render-job`, `audio-render-worker`, `audio-cleanup-worker`, `retry-admin-notification`. |
| `SUPABASE_DB_URL` | `enqueue-render-job`, `audio-render-worker`, `audio-cleanup-worker` | Postgres connection string. pg_boss requires a direct DB connection. Built-in for Edge Functions. |
| `GOOGLE_TTS_API_KEY` | `audio-render-worker` | Google Cloud Text-to-Speech API key. Created in the system administrator's GCP project. Restrict the key to the Text-to-Speech API only. Required for all `synthesizeSpeech` calls in §4.6. |
| `FCM_SERVER_KEY` | Route-change FCM dispatcher (called from `audio-render-worker`) | Firebase Cloud Messaging server credential used to send push notifications to operator devices. |
| `ADMIN_EMAIL` | Signup trigger / `retry-admin-notification` | Destination for new-operator signup notifications (§3.1). |
| `SENTRY_DSN_ANDROID` | Android app (bundled at build time, not at runtime) | Sentry project DSN for the Android tablet app. Injected into the build via Gradle build config; not stored as a Supabase secret. See §11.2. **Per-environment:** `SENTRY_DSN_ANDROID_PROD` and `SENTRY_DSN_ANDROID_STAGING` are provisioned separately and selected by Android build variant (see setup checklist preamble). |
| `SENTRY_DSN_DASHBOARD` | Next.js dashboard | Sentry project DSN for the dashboard. Configured in Vercel environment variables. See §11.2. **Per-environment:** `SENTRY_DSN_DASHBOARD_PROD` (Vercel production env) and `SENTRY_DSN_DASHBOARD_STAGING` (Vercel preview env). |
| `SENTRY_DSN_EDGE` | All Edge Functions | Sentry project DSN for Edge Functions. Set as a Supabase secret. See §11.2. **Per-environment:** `SENTRY_DSN_EDGE_PROD` (production Supabase project secret) and `SENTRY_DSN_EDGE_STAGING` (staging Supabase project secret). |

**Setup checklist** (one-off, performed by the system administrator before the first
deployment):

**Provision two Supabase projects before any step below: one for production, one for
staging.** Vercel preview deploys (every PR / branch push) target staging; the Vercel
main-branch deploy targets production. Both Supabase projects need the full setup that
follows — custom access token hook registration (item 4), pg_boss schema install (item 3),
scheduled functions (item 5), FCM credentials (item 2). Sentry is provisioned with **one
DSN per environment per surface**, so the full DSN list is six values total —
`SENTRY_DSN_ANDROID_PROD`, `SENTRY_DSN_ANDROID_STAGING`, `SENTRY_DSN_DASHBOARD_PROD`,
`SENTRY_DSN_DASHBOARD_STAGING`, `SENTRY_DSN_EDGE_PROD`, `SENTRY_DSN_EDGE_STAGING` — so
Sentry's quota and event streams stay separated by environment. Google Cloud TTS may share
one API key across environments or use two separate keys; **recommended: two separate
keys** for accounting clarity (TTS spend attributable to production vs staging) and
per-environment quota isolation. Without staging, dashboard preview deploys would hit
production data — a hard blocker for safe schema and design iteration.

1. Create a Google Cloud project; enable the Cloud Text-to-Speech API; create an API key;
   restrict it to the Text-to-Speech API. Store as `GOOGLE_TTS_API_KEY` via
   `supabase secrets set`.
2. Configure FCM credentials (server key) per existing setup; store as `FCM_SERVER_KEY`.
3. Run the one-off `pgboss-install` Edge Function to create the `pgboss` schema and tables.
4. **Register the custom access token hook (Supabase Auth).** After running the database
   migration that creates the `custom_access_token_hook` function (§3.4a), navigate to
   **Supabase Dashboard → Authentication → Hooks → Custom Access Token Hook** and select
   the `custom_access_token_hook` function. Save. **Verify** by signing in a test user
   (dashboard signup or test device pairing) and inspecting the issued JWT (e.g. via
   <https://jwt.io>): the payload must contain an `operator_id` claim (and for device
   tokens, also `device_id`). The hook is a Supabase Auth configuration setting, **not**
   a database object — it does not travel in SQL migrations and must be set manually on
   every fresh environment (production, staging, local Supabase, recovery environments).
   Absence of the hook causes silent RLS failures: every RLS-scoped query returns an
   empty result set for valid-looking JWTs because the policies' `(auth.jwt()->>'operator_id')`
   resolves to NULL. This is by far the most common cause of an apparently-broken-but-not-erroring
   fresh deployment.
5. Schedule `audio-render-worker` (`* * * * *`), `audio-cleanup-worker` (`0 3 * * *`),
   and **`pgboss-maintain` (`0 * * * *`)** via Supabase's scheduled functions. Add the
   daily `rate_limit_attempts` cleanup (§2.9) and the daily 60-day device
   auto-deregistration (§10) to the existing 03:00 UTC scheduled function rather than
   creating new schedules. The `pgboss-maintain` function runs `boss.maintain()` hourly
   and is a hard requirement when `audio-render-worker` is constructed with
   `{ supervise: false, schedule: false }` (the inviolable rule in §4.6) — without
   `pgboss-maintain`, pg_boss's internal archive/expiry/dead-job-recovery duties never
   run, and over time crashed jobs would remain stuck in `active`.
6. Reg 13(4) voice-frequency verification is **already complete** — see
   `spike-records/round-3/findings-tts-frequency.md` (empirical spectral analysis of
   `en-GB-Neural2-B` at LINEAR16 / 24 kHz; 66.67 % energy survives a 300 Hz hi-pass,
   formant-zone share 41.79 %, smoothed upper audible cutoff 3 451 Hz) and Compliance
   Mapping Matrix Reg 13(4). No additional pre-deployment voice verification work
   remains. If at any future point the locked voice is changed from `en-GB-Neural2-B`,
   the verification must be re-run against the new voice (this is itself a Case 1
   re-planning event per WORKFLOW.md).
7. Create three Sentry projects (Android, Dashboard, Edge Functions) **per environment**
   — six projects total across production and staging — and provision their DSNs (§11.2).
   Store `SENTRY_DSN_EDGE_PROD` and `SENTRY_DSN_EDGE_STAGING` as Supabase secrets in the
   respective Supabase projects; configure `SENTRY_DSN_DASHBOARD_PROD` and
   `SENTRY_DSN_DASHBOARD_STAGING` in Vercel (per-environment env vars — staging on
   preview deploys, production on the main-branch deploy); inject
   `SENTRY_DSN_ANDROID_PROD` / `SENTRY_DSN_ANDROID_STAGING` into the Android build config
   by build variant. This is a one-off task at Stage 2/3 start (when the Android app and
   dashboard exist); it does not block Stage 1 backend work.

**Order dependencies.** The checklist items are mostly independent but a few have hard
ordering that must be respected per environment:
- **(a)** The custom access token hook (item 4) must be registered **before** the first
  dashboard signup or device pairing in that environment. Without it, no JWT carries an
  `operator_id` claim and every RLS-protected query silently returns zero rows — the
  most common cause of an apparently-broken-but-not-erroring fresh deployment, as
  flagged in item 4 itself.
- **(b)** FCM credentials (item 2) must be present **before** the first
  `audio-render-worker` deployment in that environment, because the worker dispatches FCM
  on every successful render (§4.6 / §7.6) and would fail every job otherwise.
- **(c)** Sentry DSN provisioning (item 7) must precede any build that initialises Sentry
  — i.e. before the first Android release build of a given variant, and before the first
  Vercel deploy of a given environment. A build that initialises Sentry against an empty
  DSN fails open (the SDK no-ops) but loses the events that would have been the first
  signal of a deployment problem.

The other items (Google Cloud TTS, pg_boss install, scheduled function registration) are
order-independent relative to each other but must all be complete before the first
production traffic.
