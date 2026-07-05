> **HISTORICAL RECORD — NOT CURRENT AUTHORITY**
>
> This is the original **Data Architecture** for the Passenger Display System — the Supabase schema and RLS model, Edge Functions, Room (on-device) schema, sync sequence, and audio-render pipeline the product was designed and built against. It is frozen at v3.9 (May 2026).
>
> It is retained as the historical architecture record — **not** the authority for present-day design decisions. Current project state and remaining work live in `pds-planning/living/STATE.md`; the narrative history up to the July 2026 workflow migration is in `pds-planning/historical/PROJECT-HISTORY.md`. Several parts of the built system have since diverged from this document (some divergences are recorded in code comments and in PROJECT-HISTORY.md); where this document and the current implementation differ, that divergence is expected and the current implementation is authoritative.

# Data Architecture Document
# Passenger Display System (PDS)

**Version:** 3.7
**Last Updated:** May 2026
**PRD Alignment:** PRD v3.7

## Changelog

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
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Server-assigned via trigger on every INSERT or UPDATE. Used as the sync cursor. |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT false | Soft delete flag. Deleted routes remain for sync propagation. |

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

**Path scheme:**

| Path | Content |
|---|---|
| `{operator_id}/{route_id}/route_announcement.mp3` | "This bus is the [Route Name] service to [Final Stop]." |
| `{operator_id}/{route_id}/stop_{stop_order}.mp3` | "Next stop: [Stop Name]." for stop at position N |

Stop order values match `route_stops.stop_order` (0-based). A route with 10 stops produces 11 files:
one route announcement plus `stop_0.mp3` through `stop_9.mp3`.

**Access model:** Storage RLS policies scope read access by operator_id. A device JWT may read
files only under `{their_operator_id}/`. Dashboard users may read and write files under their
own `{operator_id}/`. Write operations are performed by the `render-route-audio` Edge Function
running with service-role access.

**Audio format:** MP3, mono, 32kbps. Sufficient quality for spoken announcements; ~20KB per
5-second file.

**Storage size estimate:** A 10-stop route produces approximately 220KB of audio. A 50-stop
route produces approximately 1MB. Audio file sync adds negligible bandwidth relative to the
route data itself.

**Lifecycle:**
- **Created:** When the dashboard saves a route, it calls `render-route-audio` (§4.6) asynchronously.
  Files appear in Storage when rendering completes, which typically takes seconds to low minutes.
- **Updated:** Any route save (even a minor edit) triggers a full re-render. Existing files are
  overwritten. Tablets re-download all files for that route on next sync, because the route's
  `updated_at` changed.
- **Deleted:** When a route is hard-deleted from Storage (during data retention cleanup — see §10),
  the corresponding `{operator_id}/{route_id}/` folder is removed from the bucket.

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

**Audio rendering:** After calling this RPC successfully, the dashboard also calls the
`render-route-audio` Edge Function (§4.6) asynchronously to render and store the audio files
for the saved route. The RPC itself is synchronous and completes before rendering begins; the
route is immediately available in the dashboard while audio renders in the background.

### 4.6 render-route-audio

**Caller:** dashboard (after a successful `replace_route_with_stops` call), called asynchronously
**Purpose:** render route-specific audio files server-side and store them in Supabase Storage
**Type:** Edge Function (not a Postgres RPC — requires calling an external TTS API)

**Input:** `{ route_id: string }`

**Behaviour:**
1. Read the route row and all route_stops for `route_id` from Supabase, ordered by `stop_order`.
2. Resolve the operator_id from the route row (for the Storage path).
3. Call a server-side TTS API (e.g., Google Cloud Text-to-Speech, Amazon Polly) with a single
   configured voice to render:
   - The route announcement text: "This bus is the [route.name] service to [last_stop.stop_name]."
   - For each stop N: the next-stop text: "Next stop: [stop.stop_name]."
4. Write each rendered MP3 to the `route-audio` bucket at
   `{operator_id}/{route_id}/route_announcement.mp3` and
   `{operator_id}/{route_id}/stop_{stop_order}.mp3`. Overwrite any existing files.
5. Return `{ files_written: number }`.

**Error handling:** If the TTS API call or Storage write fails, the function returns an error.
The dashboard may log this but does not surface it to the operator as a blocking error — the route
is already saved. The route will show as "Audio not ready" on tablets until a subsequent render
succeeds. A retry mechanism (manual re-save of the route, or an admin-triggered re-render) resolves
transient failures.

**Fixed announcements are not rendered here.** Termination, hail-and-ride start/end, diversion
start/end, and the alert chime are bundled in the APK and do not vary by route. This function
renders only the route-specific files.

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
| updated_at_utc | LONG | NOT NULL | Epoch millis of server timestamp |
| is_deleted | BOOLEAN | NOT NULL | Soft delete flag from Supabase. When true, hidden from route list. |
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

**File layout:**
```
audio/
  {operator_id}/
    {route_id}/
      route_announcement.mp3
      stop_0.mp3
      stop_1.mp3
      ...
      stop_N.mp3
```

**Bundled assets** (termination, H&R, diversion, alert chime, keepalive) remain in APK
`res/raw/`; they are not duplicated in `filesDir`.

**Journey-start gating:** Before enabling the "Start Journey" button for a route, the app checks
whether all expected audio files exist in `filesDir`. Expected files are derived from the route's
stop count (one `route_announcement.mp3` + one `stop_{N}.mp3` per stop). If any file is absent,
the route shows a "Audio not ready — syncing" indicator and cannot be started. This is a clear
error state; there is no fallback to on-device TTS.

**Cleanup:** When a route's `is_deleted = true` is applied locally (during sync or at journey
end for pending-deletion routes), the app deletes the corresponding `{route_id}/` directory
from local audio storage. This prevents audio files from accumulating for deleted routes.

**No Room schema change:** Audio file presence is determined by file-system stat at runtime.
No audio-file metadata table is needed.

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
7. **Download audio files (new):** For each route UPSERT'd in step 3 (not deleted), download any missing or outdated audio files from Supabase Storage to local file storage (§6.4):
   - Compute expected file list: `route_announcement.mp3` + `stop_{N}.mp3` for each stop (N = 0 to stop_count − 1).
   - If the route was updated in this sync (i.e., it appears in the `get_routes_since` response), re-download all audio files for that route, overwriting any existing local files. This handles stop renames and route name changes.
   - If a file is new (did not exist locally), download it.
   - Audio download failures are non-fatal: log `AUDIO_FILE_MISSING` to journey_events, continue sync. The route will show "Audio not ready" until the next successful download.
8. Update the device's `last_seen_at` in Supabase (separate query).

### 7.4 Sync Algorithm — Full Sequence

1. Set `sync_metadata.sync_status = 'syncing'`.
2. Check operator account status by querying the `operators` row. Abort sync if `status != 'active'`:
   - `status = 'pending'`: display "Account pending approval — contact your administrator" and abort.
   - `status = 'suspended'`: display "Account Suspended — please contact your bus company administrator" and abort.
3. **Download** remote route and stop changes per section 7.2 steps 1–6.
4. **Download audio files** per section 7.2 step 7. Audio download runs after route data is committed to Room. Audio failures do not fail the sync — routes are updated even if audio files are temporarily unavailable.
5. Set `sync_metadata.sync_status = 'synced'`.
6. Update `devices.active_route_id` if a journey is in progress.
7. On any failure in steps 1–3 or 5, set `sync_status = 'failed'` and retry on next trigger. Audio download failure (step 4) is logged but does not set `sync_status = 'failed'`.

### 7.5 Conflict Resolution

In the initial release, conflicts cannot occur because tablets are read-only for routes. Only the dashboard writes routes, and only one dashboard user exists per operator (FR-WD-06).

### 7.6 FCM Push Notifications

When a route is inserted, updated, or deleted on Supabase, a database trigger calls a Supabase Edge Function that:
1. Identifies the operator_id from the changed route.
2. Queries `devices` for all active devices belonging to that operator.
3. Sends an FCM message to each device's FCM token, with a simple payload signalling "routes have changed."

Tablets register their FCM registration token in `devices.fcm_token` (see §2.2) during the first successful sync after pairing. The token is re-registered whenever Firebase rotates it. On receiving an FCM message, the tablet triggers an immediate sync.

FCM is the mechanism that enables sub-5-minute route propagation (PRD §12 success metric) for tablets that are already online. If FCM is unavailable (device offline, FCM service down), the connectivity-change and 30-minute periodic triggers ensure eventual consistency — propagation may take up to 30 minutes in the worst case.

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

### 8.4 Route Sync Flow

    Dashboard      Supabase / Storage   render-route-audio    FCM            Tablet
       |                  |                    |               |               |
       |-- Save route --->|                    |               |               |
       |                  |-- replace_route_with_stops         |               |
       |                  |-- Trigger: routes.updated_at       |               |
       |                  |-- Trigger: notify FCM              |               |
       |                  |--------------------------------->  |               |
       |                  |   [async] --> render-route-audio --+               |
       |                  |                    |-- Render route_announcement.mp3
       |                  |                    |-- Render stop_N.mp3 (per stop)
       |                  |                    |-- Write to route-audio bucket  |
       |                  |                                    |               |
       |                  |                               |----|               |
       |                  |                               |--> Push to device tokens
       |                  |                                                    |
       |                  |<-- get_routes_since(last_ts) ----------------------|
       |                  |--------- Return changed routes -------------------->
       |                                                                       |-- UPSERT routes
       |                                                                       |-- Replace route_stops
       |                                                                       |-- Update last_server_timestamp
       |                  |<-- Storage: fetch audio files ---------------------|
       |                  |--------- Stream MP3 files ------------------------->
       |                                                                       |-- Store in filesDir/audio/

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
| Soft-deleted routes | Indefinite in Supabase | Could be hard-deleted after 90 days (operations decision) |
| Journey state (local) | Cleared on journey end | is_active = false when journey completes; pending_deletion cleanup runs here too |
| Inactive devices | Indefinite in Supabase | Could be auto-deregistered after extended inactivity (operations decision) |

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
calls `File.exists()` on each expected audio file (one route announcement + one per stop). If any
file is missing, the route renders with an "Audio not ready — syncing" indicator and the button is
disabled. This is the only error state for missing audio; there is no fallback to on-device TTS.
On-device TTS is not used anywhere in this product.

**Audio file outdating.** When a route appears in the `get_routes_since` response (i.e., it was
updated since the last sync), all audio files for that route are re-downloaded regardless of
whether they already exist locally. This handles stop renames and route name changes without
requiring a separate audio-version tracking field — the route's own `updated_at` serves as the
version signal.

**Audio download is non-fatal.** A failed audio file download does not set `sync_status = 'failed'`
and does not prevent the route data from being written to Room. The route is usable once audio
files are present; the "Audio not ready" indicator disappears after the next successful download.
