# Data Architecture Document
# Passenger Display System (PDS)

**Version:** 3.8
**Last Updated:** May 2026
**PRD Alignment:** PRD v3.8

## Changelog

### v3.8 (May 2026)
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
| status | TEXT | NOT NULL, DEFAULT 'active' | 'active' or 'inactive'. Set to 'inactive' by the dashboard's "Deactivate device" action or by `Deregister Device` on the tablet. Inactive devices do not count toward billing. |
| registered_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | When the device first paired |
| last_seen_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Updated on every successful sync and by the heartbeat mechanism (§7.7). Drives online/offline status in the dashboard. A device is considered online if this timestamp is within the last 5 minutes. |
| fcm_token | TEXT | NULLABLE | The FCM registration token for this device. Set during the first successful sync after pairing, when the tablet registers with Firebase and stores the token here. Updated automatically when Firebase rotates the token. Nullable for devices that have not yet completed their first sync. Used by the route-change Edge Function (§7.6) to deliver push notifications. |
| audio_enabled | BOOLEAN | NOT NULL, DEFAULT true | Whether this device produces audio output. `true` for the designated primary (audio) tablet; `false` for display-only tablets. Default `true` for the first tablet paired to an operator (set by `pair-device` based on existing device count — see §4.2 step 6a); `false` for subsequent tablets. Configurable from the dashboard fleet view. Tablets read this value from their device record after each sync and apply it immediately: if `false`, all audio output (announcements, alert chime, Bluetooth keep-alive) is suppressed for the session; visual display continues normally. |
| active_route_id | UUID | NULLABLE, FK → routes(id) ON DELETE SET NULL | The route currently active on this device. Updated by the tablet on journey start/end. Used by the dashboard's fleet view. Nullable when no journey is in progress. `ON DELETE SET NULL` ensures that if a route row is hard-deleted from Supabase (see §10 data retention), this FK reference becomes null rather than causing a constraint violation or cascading device deletion. |

**Indexes:** `user_id`; `(operator_id, status)` composite; `android_id` (for recovery lookups); `fcm_token` (for push dispatch lookups).

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
| last_synced_with_return | TIMESTAMPTZ | NULLABLE, DEFAULT NULL | Timestamp set on both routes when a return route is generated (FR-WD-12). Used by the dashboard to detect divergence: if `updated_at > last_synced_with_return` and `return_route_id IS NOT NULL`, the dashboard warns the operator that the linked return route may now be divergent and offers to regenerate it. NULL if no return has ever been generated for this route. |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Server-assigned via trigger on every INSERT or UPDATE. Used as the sync cursor and as the `route_version` segment of the Storage path scheme (§2.7), expressed as epoch milliseconds. |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT false | Soft delete flag. Deleted routes remain for sync propagation. |
| audio_render_status | TEXT | NOT NULL, DEFAULT 'pending', CHECK (audio_render_status IN ('pending', 'ok', 'failed')) | Current state of the audio render job for this route version. `pending` immediately after `replace_route_with_stops`. Flipped to `ok` by the audio render worker on successful completion (§4.6); flipped to `failed` after the worker exhausts its retries. The dashboard surfaces this status in the route list (PRD FR-WD-13); the tablet uses it to skip audio download for failed routes (§7.4). |
| audio_render_error | TEXT | NULLABLE | Last error message captured when `audio_render_status = 'failed'`. NULL when status is `pending` or `ok`. Cleared back to NULL by the worker on a successful re-render. Surfaced on the dashboard for diagnostic purposes. |
| audio_announcement_hash | TEXT | NULLABLE | SHA-256 hex of the route-announcement text (`"This bus is the [route.name] service to [last_stop.stop_name]."`) that was rendered into the currently-stored `route_announcement.mp3`. Compared by the audio render worker against the would-be hash on re-render to skip rendering when unchanged. NULL until the first successful render of this route. See §4.6 for the differential re-render algorithm. |

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
| audio_content_hash | TEXT | NULLABLE | SHA-256 hex of the per-stop announcement text (`"Next stop: [stop.stop_name]."`) that was rendered into the currently-stored `stop_{stop_order}.mp3` for this stop. Used by the audio render worker to skip re-rendering when the stop's text has not changed (§4.6 differential re-render). NULL until this stop has been successfully rendered at least once. |

**Indexes:** `route_id` (for loading all stops for a route).

**No updated_at or is_deleted:** Stops do not have their own sync timestamps or soft-delete flags. When a route is synced, its entire stop list is replaced atomically by the `replace_route_with_stops` RPC.

### 2.6 naptan_stations

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

### 2.7 Supabase Storage — Route Audio Files

Pre-rendered audio files for route-specific announcements are stored in Supabase Storage. Fixed
announcement texts that do not vary by route (termination, hail-and-ride start/end, diversion
start/end) are bundled in the APK and are not stored here.

**Bucket:** `route-audio` (private)

**Path scheme (version-keyed):**

| Path | Content |
|---|---|
| `{operator_id}/{route_id}/{route_version}/route_announcement.mp3` | "This bus is the [Route Name] service to [Final Stop]." |
| `{operator_id}/{route_id}/{route_version}/stop_{stop_order}.mp3` | "Next stop: [Stop Name]." for stop at position N |

`route_version` is `routes.updated_at` expressed as epoch milliseconds (e.g. `1715846400000`).
Each route save produces a freshly-stamped `updated_at` (via the trigger in §3.5) and therefore
a fresh path prefix. Stop order values match `route_stops.stop_order` (0-based). A 10-stop route
produces 11 files per version: one route announcement plus `stop_0.mp3` through `stop_9.mp3`.

**Race-safety by construction:** Two concurrent saves cannot collide on the same Storage path
because each save yields a different `updated_at`. The newer save enqueues its own render job;
the older job's worker detects staleness on dequeue (§4.6 job-processing step 1) and exits
without rendering.

**Access model:** Storage RLS policies scope read access by operator_id. A device JWT may read
files only under `{their_operator_id}/`. Dashboard users may read files under their own
`{operator_id}/`. Write operations are performed by the `audio-render-worker` Edge Function
(§4.6) running with service-role access.

**Audio format:** MP3, mono, 22.05kHz sample rate, ~32kbps. Sufficient quality for spoken
announcements; ~20KB per 5-second file.

**Storage size estimate:** A 10-stop route produces approximately 220KB of audio per version.
With the two-version retention policy (§4.7), the steady-state Storage footprint per route is
approximately 2× a single version. Audio file sync adds negligible bandwidth relative to the
route data itself.

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
  the two most-recent versions per route.
- **Deleted:** When a route is hard-deleted (data retention cleanup, §10), the entire
  `{operator_id}/{route_id}/` folder — every version — is removed from the bucket.

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

### 3.3 Refresh Token Recovery

If a tablet sits offline long enough that its refresh token expires (typical Supabase default is 7 days; we extend this in project settings to allow weeks/months), the next sync attempt returns a 401. The tablet detects this and silently calls the `recover-device` Edge Function:

1. The tablet passes its stored Android ID and plaintext device secret. (No JWT — the device is unauthenticated at this point.)
2. The function looks up the `devices` row by `android_id`. Rejects with HTTP 404 if not found.
3. Computes the SHA-256 hash of the provided `device_secret` and compares against the stored `device_secret_hash`. Rejects with HTTP 401 if the hashes do not match. (Uses constant-time comparison to avoid timing attacks.)
4. Rejects if `devices.status = 'inactive'` or if the linked operator has `status != 'active'`.
5. Issues a fresh session for the existing anonymous user (`user_id` on the devices row) via the Admin API. The custom access token hook (§3.4a) fires automatically and stamps `operator_id` and `device_id` claims.
6. Returns `{ access_token, refresh_token, device_id, operator_id }`.

The device row is unchanged; only the auth session is refreshed. The user does not see the pairing screen again.

If `recover-device` fails (device deactivated, secret mismatch, row missing), the tablet shows a clear message and returns to the first-run setup screen for re-pairing.

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
3. Create a new anonymous Supabase Auth user via the Admin API, setting `app_metadata: { role: 'device', operator_id: '<UUID>' }` (device_id is added in step 6 once the devices row UUID is known).
4. Generate a cryptographically random 256-bit device secret: `crypto.getRandomValues(new Uint8Array(32))`, base64url-encoded (43 chars).
5. Compute the SHA-256 hash of the plaintext secret (hex-encoded string).
6. Insert a `devices` row with the new `user_id`, `operator_id` from the code, the `android_id` from the request, the `device_secret_hash`, and default `display_name` "New Device #N" where N is one more than the operator's current device count.
6a. Set `audio_enabled` on the new devices row: query the count of existing active (non-newly-inserted) devices for this operator. If count = 0, set `audio_enabled = true` (this is the first tablet — it is the default primary). If count ≥ 1, set `audio_enabled = false`. This implements the default audio designation without requiring fleet manager action for single-tablet deployments.
7. Update the anonymous user's `app_metadata` to add `device_id` (the UUID of the newly created devices row).
8. Mark the pairing code as used (`used_at = now()`).
9. Create a session for the anonymous user via the Admin API. The custom access token hook (§3.4a) fires automatically and stamps `operator_id` and `device_id` claims.
10. Return `{ access_token, refresh_token, device_id, operator_id, operator_name, device_secret }`. The `device_secret` is the plaintext value — included in this response only. The server does not log or retain it after the response is sent.

Failure modes are handled cleanly: invalid code returns a clear error message; transient errors trigger a retry on the tablet.

**Rate limiting:** To prevent brute-force enumeration of the 6-digit pairing code space, `pair-device` enforces:
- **Per-IP:** max 10 attempts per 10-minute window. Tracked in a `rate_limit_attempts` table (or Edge Function KV store) keyed by IP + function name + window bucket.
- **Exponential backoff:** after 3 consecutive failures from the same IP within 60 seconds, subsequent attempts receive HTTP 429 with a `Retry-After` header. Back-off intervals: 5 s, 15 s, 60 s for attempts 4, 5, 6+.
- Failures against valid-but-already-used or expired codes count toward the limit; failures against genuinely invalid codes count at double weight.
- Rate limit state resets when a new 10-minute window begins.

### 4.3 recover-device

**Caller:** tablet (unauthenticated; calls with anon key)
**Purpose:** silently re-issue a session for an existing device after refresh token expiry

**Input:** `{ android_id: string, device_secret: string }`

**Behaviour:**
1. Look up the `devices` row by `android_id`. Reject with HTTP 404 if not found.
2. Compute the SHA-256 hash of the provided `device_secret`. Compare against the stored `device_secret_hash` using a constant-time comparison. Reject with HTTP 401 if the hashes do not match.
3. Reject if `devices.status = 'inactive'` or if the linked operator has `status != 'active'`.
4. Issue a fresh session for the device's `user_id` via the Admin API. The custom access token hook (§3.4a) fires automatically and stamps `operator_id` and `device_id` claims.
5. Return `{ access_token, refresh_token, device_id, operator_id }`.

**Rate limiting:** `recover-device` is an unauthenticated endpoint. To limit brute-force attempts against the device secret:
- **Per-Android-ID:** max 5 attempts per hour. After 3 consecutive failures (wrong secret or inactive device), lock out that Android ID for 15 minutes.
- **Per-IP:** max 20 attempts per hour across all Android IDs from that IP.
- Both limits tracked in the `rate_limit_attempts` table or Edge Function KV store.
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

The server's `updated_at` trigger fires automatically on the route UPSERT.

This RPC is what the dashboard calls when saving a route.

**Audio rendering:** After calling this RPC successfully, the dashboard calls the
`enqueue-render-job` Edge Function (§4.6) which enqueues a `render-route-audio` job onto the
pg_boss queue. The dashboard does not call any synchronous render API and does not wait for
the render to complete. The route appears in the route list immediately with
`audio_render_status = 'pending'`; the status flips to `ok` (or `failed`) when the
`audio-render-worker` processes the job. The dashboard does **not** fire any FCM push on save —
FCM dispatch is the render worker's responsibility on successful completion (§4.6 and §8.4).

### 4.6 Audio Render Job (pg_boss queue + scheduled worker)

Audio rendering runs as a **pg_boss-backed job**, not as a synchronously-invoked Edge Function.
This section specifies the queue, the enqueue path, the worker, and the per-job processing
algorithm. The job replaces the previous `render-route-audio` Edge Function (removed in v3.8);
the *output* is unchanged (pre-rendered MP3 files in Supabase Storage) but the production
mechanism is asynchronous, retryable, idempotent, and observable.

**Why pg_boss.** A Postgres-backed job queue gives us durability, retry, and backoff with no
new infrastructure: pg_boss creates its own tables inside the existing Supabase Postgres
database. There is no separate queue service to deploy or pay for. The trade-off — the worker
runs as a scheduled Edge Function rather than a long-running consumer — is acceptable for the
initial release's expected queue depth (a single operator may save a handful of routes per
week; even at 100 operators that is tens of jobs per day).

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
2. Connect to pg_boss via the Supabase Postgres connection (`SUPABASE_DB_URL`).
3. Call `boss.send('render-route-audio', { route_id, route_version })`.
4. Return `{ job_id }`.

Direct client-side enqueue is avoided because pg_boss writes require Postgres credentials that
must not be exposed to the browser. `enqueue-render-job` is deliberately thin — it does not
read the route data itself; that is the worker's job.

The dashboard also offers a "Re-render audio" action per route (PRD FR-WD-21) which calls
`enqueue-render-job` with the route's current `updated_at` as `route_version` — useful for
recovering from terminal failures without modifying route data.

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
   - Define `target_path = {operator_id}/{route_id}/{route_version}/<file>.mp3`.
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
     - `audioConfig: { audioEncoding: 'MP3', sampleRateHertz: 22050 }`
   - **The voice is locked to `en-GB-Neural2-B`.** It is not configurable per operator, per
     route, or per deployment. Changing the voice requires re-running the Reg 13(4)
     frequency verification (Compliance Mapping Matrix) and is therefore a deliberate
     re-planning event, not a runtime knob. This is an inviolable rule alongside the
     alert-chime sequence.
   - Upload the resulting MP3 bytes to Storage at `target_path` using the service-role
     Supabase client.
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
3. Keep the **two most-recent** versions. Recursively delete every older version path.

**Why two versions, not one.** Two-version retention gives a tablet a recovery window if it
synced the route metadata just before a new save: the previous version's audio files are
still available for download, so a tablet that pulled the metadata for version N-1 between
when N was saved and when N's render completes can still successfully download version N-1's
audio. One-version retention would risk a 404 in that race window.

The cleanup job is idempotent — running it more than once a day is harmless because each run
recomputes the keep-set independently. Failure within a day is logged but not retried inside
the same day; the next day's run picks up the missed work.

Hard-deletion of a route (data retention cleanup, §10) overrides this policy and removes all
versions for that route.

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
| updated_at_utc | LONG | NOT NULL | Epoch millis of server timestamp. Also used as the `route_version` path segment for audio downloads (§7.2 step 7, §2.7). |
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
| event_type | TEXT | NOT NULL | JOURNEY_START, JOURNEY_END, STOP_ANNOUNCED, STOP_PASSED_WITHOUT_DETECTION, STOP_SKIPPED, HAIL_AND_RIDE_SECTION_STARTED, HAIL_AND_RIDE_SECTION_ENDED, DIVERSION_STARTED, DIVERSION_ENDED, GPS_LOST, GPS_REGAINED, AUDIO_DISCONNECTED, AUDIO_RECONNECTED, AUDIO_FILE_MISSING, AUDIO_PLAYBACK_ERROR, CLOCK_DRIFT, SYNC_SUCCESS, SYNC_FAILURE, APP_ERROR |
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
| journey_started_at | LONG | NOT NULL | Epoch millis when the journey began |
| is_active | BOOLEAN | NOT NULL | True during a journey. On app restart, if true, the journey is resumed. |

**Note on diversion state:** Skipped stops for an active diversion are stored in the `journey_skipped_stops` table (§5.7), not in this row, to keep the single-row design clean and support fast indexed lookup.

**Note on single-row design and multi-tablet future:** The single-row design (one active journey per device) is correct for the initial release. Each tablet is an independent device with its own journey state; there is no shared journey state between tablets in the initial release (see PRD §5.2). The PRD §5.2 gestures at a future "primary-secondary tablet linking with shared journey state" feature. That feature would require a fundamentally different architecture — journey state would need to be a Supabase Realtime-projected record rather than a local single-row table. That redesign is deliberately deferred. The single-row `journey_state` table is the correct, intentional design for the initial release scope.

### 5.6 sync_metadata (Local Only)

Tracks sync state. Single-row table.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | INTEGER | PK, hardcoded to 1 | Same single-row pattern as journey_state |
| last_sync_at | LONG | NULLABLE | Epoch millis of last successful sync completion |
| sync_status | TEXT | NOT NULL, DEFAULT 'never' | 'synced', 'syncing', 'failed', 'never' |
| last_server_timestamp | LONG | NOT NULL, DEFAULT 0 | Server transaction timestamp from the last successful download. Used as the sync cursor for `get_routes_since`. |

### 5.7 journey_skipped_stops (Local Only)

Transient list of stop indices skipped due to an active driver-initiated diversion. Not synced. Cleared at journey start and end.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | INTEGER | PK, AUTO-INCREMENT | Local auto-incrementing ID |
| stop_index | INTEGER | NOT NULL, UNIQUE | The 0-based `stop_order` index of the stop to skip. UNIQUE prevents duplicate entries for the same stop. |

**Implementation note:** When `journey_state.current_stop_index` advances to a value present in this table, the GPS state machine immediately increments the index again (repeatedly, until reaching a non-skipped index) without waiting for GPS proximity and without firing any announcement. A `STOP_SKIPPED` event is logged for each skipped stop. This table is emptied (`DELETE FROM journey_skipped_stops`) at journey start and on "Diversion end." Since there is only ever one active journey at a time, no journey ID foreign key is required.

---

## 6. Non-Database Storage

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
| alert_chime.mp3 | MP3 | ~50KB | Alert tone before termination, diversion, and hail-and-ride announcements |
| silent_keepalive.mp3 | MP3 | ~10KB | Inaudible 1-second audio for Bluetooth speaker keepalive |
| termination.mp3 | MP3 | ~50KB | Pre-rendered fixed announcement: "This service terminates here. All change, please." |
| hail_and_ride_start.mp3 | MP3 | ~60KB | Pre-rendered fixed announcement: "You are now entering a hail and ride section. Please signal the driver if you wish to alight." |
| hail_and_ride_end.mp3 | MP3 | ~50KB | Pre-rendered fixed announcement: "You are now leaving the hail and ride section." |
| diversion_start.mp3 | MP3 | ~55KB | Pre-rendered fixed announcement: "This service is on diversion. Please check the display for affected stops." |
| diversion_end.mp3 | MP3 | ~50KB | Pre-rendered fixed announcement: "This service has resumed its normal route." |

The five announcement files are rendered once using the same server-side TTS voice as
`render-route-audio` (§4.6), ensuring consistent voice quality between fixed and route-specific
audio. They are updated by shipping a new APK version — not by Supabase Storage sync. Specific
affected stops are not named in the diversion announcement audio; they are conveyed visually via
the tube-map display (see Compliance Mapping Matrix Reg 10(1)).

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
        route_announcement.mp3
        stop_0.mp3
        stop_1.mp3
        ...
        stop_N.mp3
```

`{route_version}` matches the version segment in the Storage path scheme (§2.7) and is the
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
   Expected files are derived from the route's stop count (one `route_announcement.mp3` +
   one `stop_{N}.mp3` per stop).

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
7. **Download audio files (version-keyed):** For each route UPSERT'd in step 3 (not deleted), download audio files from Supabase Storage to local file storage (§6.4) using the route's version-keyed path scheme (§2.7):
   - **Render-status guard.** If the route's `audio_render_status = 'failed'`, do **not** attempt any audio download for it — the version-keyed path will not exist in Storage and the request would 404. Log `AUDIO_FILE_MISSING` once with `detail = 'render_failed'` and move on. The "Audio not ready" indicator (§6.4) continues to gate journey starts. If `audio_render_status = 'pending'`, also defer download — a future sync (after the render worker completes) will see `ok`.
   - **For routes with `audio_render_status = 'ok'`:** compute `route_version` as the route's `updated_at` in epoch millis. The expected file list is `route_announcement.mp3` plus `stop_{N}.mp3` for each stop, all under `{operator_id}/{route_id}/{route_version}/`.
   - If the route was updated in this sync (it appears in the `get_routes_since` response), the path's `route_version` segment has changed; download all audio files for the new version into a fresh local `{route_id}/{route_version}/` directory (see §6.4 layout note). The previous local version directory is removed once the new download completes successfully.
   - If a file is missing locally for the current version, download it.
   - The tablet pulls audio for the specific route version it just synced; older or newer Storage versions are not pulled. If the dashboard saves the route again while the tablet is mid-download, the tablet completes its download of the version it asked for; the next sync round picks up the newer version.
   - Audio download failures (network drop, 404 on a partially-rendered version) are non-fatal: log `AUDIO_FILE_MISSING` to journey_events, continue sync. The route shows "Audio not ready" until the next successful download.
8. Update the device's `last_seen_at` in Supabase (separate query).

### 7.4 Sync Algorithm — Full Sequence

1. Set `sync_metadata.sync_status = 'syncing'`.
2. Check operator account status by querying the `operators` row. Abort sync if `status != 'active'`:
   - `status = 'pending'`: display "Account pending approval — contact your administrator" and abort.
   - `status = 'suspended'`: display "Account Suspended — please contact your bus company administrator" and abort.
3. **Download** remote route and stop changes per section 7.2 steps 1–6. The route rows pulled by `get_routes_since` carry the new `audio_render_status` and `audio_render_error` columns (§2.4); these are written into the local Room `routes` mirror (§5.1) and used by the audio-download step below.
4. **Download audio files** per section 7.2 step 7, using version-keyed paths derived from each route's `updated_at` and skipping any route whose `audio_render_status` is `failed` or `pending`. Audio download runs after route data is committed to Room. Audio failures do not fail the sync — routes are updated even if audio files are temporarily unavailable.
5. Set `sync_metadata.sync_status = 'synced'`.
6. Update `devices.active_route_id` if a journey is in progress.
7. On any failure in steps 1–3 or 5, set `sync_status = 'failed'` and retry on next trigger. Audio download failure (step 4) is logged but does not set `sync_status = 'failed'`.

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
1. Takes a `route_id` as input.
2. Queries `routes` to resolve `operator_id`.
3. Queries `devices` for all active devices belonging to that operator with a non-null
   `fcm_token`.
4. Sends an FCM message to each device's FCM token with a simple payload signalling "routes
   have changed."

The dispatcher shape is unchanged from earlier versions; only the call site has moved (from
a DB trigger on `routes.updated_at` to the audio render worker's success branch).

Tablets register their FCM registration token in `devices.fcm_token` (see §2.2) during the
first successful sync after pairing. The token is re-registered whenever Firebase rotates it.
On receiving an FCM message, the tablet triggers an immediate sync.

FCM is the mechanism that enables sub-5-minute route propagation **from successful render**
(PRD §12 success metric — note that the metric now starts the clock at render completion,
not at dashboard save, because FCM no longer fires at save). If FCM is unavailable (device
offline, FCM service down), the connectivity-change and 30-minute periodic triggers ensure
eventual consistency — propagation may take up to 30 minutes in the worst case.

### 7.7 Heartbeat Mechanism

The heartbeat is a lightweight periodic update that keeps `devices.last_seen_at` current whenever the tablet has connectivity. It is independent of route sync: it does not fetch routes, process changes, or update any other state. It runs even when there is nothing to sync and during active journeys.

**Purpose:** The dashboard's fleet view uses `last_seen_at` to show online/offline status. Without a heartbeat, `last_seen_at` only updates when a route sync occurs (every 30 minutes at most), causing healthy in-service tablets to appear offline. The heartbeat ensures `last_seen_at` is accurate throughout the device's operating day.

**Interval:** 2 minutes.

**Online threshold:** A device is considered online if `last_seen_at` is within the last 5 minutes. The 2-minute heartbeat interval plus a 3-minute margin for network hiccups gives a 5-minute threshold that a healthy tablet reliably meets.

**Implementation — two-path:**

1. **During active journeys (foreground GPS service):** A `Handler.postDelayed` loop with a 2-minute interval runs inside the foreground GPS service. Because the GPS service is a foreground service (with persistent notification), it is protected from OEM battery optimisation kills that routinely throttle WorkManager on cheap Android hardware. This is the critical path: it ensures fleet status is accurate during the exact window a bus is in service.

2. **When idle / backgrounded (WorkManager):** A `PeriodicWorkRequest` with a 2-minute interval handles heartbeats when no journey is active and the GPS service is not running. WorkManager may be delayed by OEM scheduling on aggressive devices, but the consequence during idle periods (slightly stale `last_seen_at`) is acceptable.

**Database operation:** A single `UPDATE devices SET last_seen_at = now() WHERE id = :device_id` query using the device's existing Supabase session. No payload, no route data.

**Failure handling:** Heartbeat failures are silent. No UI indication, no retry. The next successful heartbeat updates the timestamp. A failed heartbeat does not affect journey operation.

**OEM best-effort caveat (idle path):** The WorkManager `PeriodicWorkRequest` used for idle-path heartbeats is best-effort on hostile-OEM cheap tablets — the same OEM category flagged in the PRD risk table for foreground service killing. Aggressive OEM battery management can throttle or delay WorkManager tasks in ways that `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` does not address. As a result: **the journey-path heartbeat (Handler loop inside the foreground GPS service) is reliable and protected**; the **idle-path WorkManager heartbeat is not guaranteed on hostile hardware**. A tablet that is powered off, in aggressive doze mode, or managed by a hostile OEM when idle may show offline in the fleet view until its next boot or until it starts a journey (at which point the protected foreground service takes over). This is acceptable and documented behaviour — the critical online-status window is during active journeys, which the foreground-service path reliably covers. Fleet managers should interpret idle tablets as "not in service" rather than "connectivity failure."

**Relationship to sync:** The heartbeat does not replace or interact with the three sync triggers (§7.1). Sync remains triggered by ConnectivityManager, FCM, and the 30-minute WorkManager periodic task. The heartbeat only updates `last_seen_at`.

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
       |                   |                       |-- Play alert_chime.mp3 then hail_and_ride_start.mp3
       |                   |                       |-- Update tube-map: H&R section rendered dashed
       |                   |
       |   [Stop N has segment_type='scheduled' AND preceding stop was 'hail_and_ride']
       |                   |-- (Segment boundary: exiting H&R section)
       |                   |-- INSERT journey_event (HAIL_AND_RIDE_SECTION_ENDED, trigger=GPS)
       |                   |-- INSERT journey_event (STOP_ANNOUNCED, stop N, trigger=GPS)
       |                   |-- Update journey_state.current_stop_index → N+1
       |                   |--------- Emit StateFlow update ---------->|
       |                   |                       |-- Play alert_chime.mp3 then hail_and_ride_end.mp3
       |                   |                       |-- Play stop_{N}.mp3
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
       |                  |<-- Storage PUT at {operator_id}/{route_id}/{route_version}/...mp3 --------|
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
       |                  |<-- Storage GET {operator_id}/{route_id}/{route_version}/*.mp3 -----------|
       |                  |--- Stream MP3 files -------------------------------------------------->  |
       |                                                                                             |-- Store in filesDir/audio/{operator_id}/{route_id}/{route_version}/

    Failure branch (terminal, after pg_boss retries exhausted):
       |                  |   UPDATE routes SET audio_render_status='failed', audio_render_error=$msg
       |                  |   (FCM is NOT fired)
       |                  |   Failure surfaces on dashboard route list (FR-WD-13);
       |                  |   "Re-render audio" action (FR-WD-21) re-enqueues.
       |                  |   Tablets see the 'failed' status on their next regular sync and skip audio download.

### 8.5 Heartbeat Flow

    GPS Service (foreground, journey active)        Supabase
       |                                               |
       |-- [every 2 minutes, Handler.postDelayed] ---->|
       |   UPDATE devices SET last_seen_at = now()     |
       |<-- 200 OK ------------------------------------|
       |                                               |
       |   [on failure: log silently, no retry]        |

    WorkManager (idle, no journey active)           Supabase
       |                                               |
       |-- [every 2 minutes, PeriodicWorkRequest] ---->|
       |   UPDATE devices SET last_seen_at = now()     |
       |<-- 200 OK ------------------------------------|
       |                                               |
       |   [on failure: log silently, no retry]        |
       |   [next scheduled run retries naturally]      |

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
        |                                       |
        |                                       +-- (1:many) --> route_stops
        |
        +-- (1:1) --> devices                              (alternative path: tablet user)

    device_pairing_codes: standalone, short-lived. Linked to operators by FK.
    naptan_stations:      standalone reference data, shared across all operators.

    Local (Room) tables: routes, route_stops,
                         journey_events, journey_state, sync_metadata

---

## 10. Data Retention and Cleanup

| Data | Retention | Cleanup |
|---|---|---|
| Journey events (local) | 30 days | Auto-deleted on app startup |
| Used/expired pairing codes | 1 hour | Scheduled cleanup function in Supabase |
| Soft-deleted routes | Indefinite in Supabase | Could be hard-deleted after 90 days (operations decision); hard-deletion removes all audio versions for the route from Storage |
| Journey state (local) | Cleared on journey end | is_active = false when journey completes; pending_deletion cleanup runs here too |
| Inactive devices | Indefinite in Supabase | Could be auto-deregistered after extended inactivity (operations decision) |
| Route audio (Storage) | Two most-recent versions per route | Daily `audio-cleanup-worker` (§4.7) at 03:00 UTC; older `{route_version}` paths removed |
| pg_boss jobs | 7 days after completion or terminal failure | `retentionDays: 7` on the `render-route-audio` queue (§4.6); pg_boss archives to `pgboss.archive` then prunes |

---

## 11. Implementation Notes

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
rendered to the corresponding MP3 file. On re-render, the audio render worker (§4.6) computes
the would-be hashes for the current text, compares against the stored hashes, and only
renders stops whose hash has changed. Unchanged audio is server-side-copied from the previous
version's Storage path into the new version's path, so a direction-label-only edit (which
changes no announcement text) calls Google TTS zero times.

**Version-keyed Storage paths.** Each route save produces a new `{route_version}` path prefix
(`routes.updated_at` in epoch millis). Tablets pull audio for the exact route version they
synced; older versions remain in Storage for a recovery window of two versions (cleanup
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

**Setup checklist** (one-off, performed by the system administrator before the first
deployment):

1. Create a Google Cloud project; enable the Cloud Text-to-Speech API; create an API key;
   restrict it to the Text-to-Speech API. Store as `GOOGLE_TTS_API_KEY` via
   `supabase secrets set`.
2. Configure FCM credentials (server key) per existing setup; store as `FCM_SERVER_KEY`.
3. Run the one-off `pgboss-install` Edge Function to create the `pgboss` schema and tables.
4. Schedule `audio-render-worker` (`* * * * *`) and `audio-cleanup-worker` (`0 3 * * *`) via
   Supabase's scheduled functions.
5. Complete the Reg 13(4) voice-frequency verification described in the Compliance Mapping
   Matrix and record the result in the operations runbook before serving any production
   traffic.
