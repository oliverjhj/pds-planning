# CLAUDE.md
# Passenger Display System (PDS) — Android Application

**Version:** 3.9 (May 2026)

This file is your reference for working on the PDS Android application. Read it at the start of every session. It contains conventions, architectural rules, and known gotchas. If a rule here conflicts with a prompt, ask before deviating.

## Changelog

### v3.9 (May 2026)
Round-3 post-adversarial-review close-out sweep. Applies the per-prompt CLAUDE.md impact lists produced by round-3 Tasks 2 and 3. Substantive deltas — itemised in the v3.9 sweep summary at the end of this file — include: audio file format switched MP3 → **LINEAR16 WAV** end-to-end (file tree, Rule 9, Rule 10, Rule 11, Rule 12, gotchas); Reg 13(4) frequency-range now empirically verified (no longer a pre-deployment blocker); Rule 17 Sentry PII clarified — `device_id` is a permitted diagnostic UUID breadcrumb, not a banned identifier; FR-AT-50 credential wipe now explicitly enumerates JWT + refresh token + device secret (parity with FR-AT-04 terminal recovery); compound 60-day auto-deregistration (silence AND ≥5 failed recover-device attempts) replaces the simple silence threshold; new gotcha — local `audio_enabled` lives in the Room `device_state` table (§5.9 Data-Arch), not SharedPreferences; new gotcha — `journey_state.diversion_invoked_at_any_point` is a one-way latch reset only on a new journey; FR-AT-28 audio-not-ready screen now shows the "Contact your fleet manager…" driver hint.

### v3.8 (May 2026)
Previously unversioned. Brought into sync with the v3.8 planning suite via the round-2 close-out sweep. Reflects all round-1 (PRD/Data-Arch/Compliance v3.0 → v3.7) and round-2 (v3.7 → v3.8) decisions that had never propagated into CLAUDE.md. Substantive deltas — itemised in "Sweep summary" at the end of this file — include: bundled tablet NaPTAN removed; upload-sync scaffolding removed; Kiosk Level 2 deferred (Level 1 is the initial release); on-device TTS forbidden, pre-rendered audio pipeline documented; per-stop proximity radius + two-stop look-ahead + GPS accuracy gate; `audio_render_status` journey-start gate; per-device `audio_enabled` with honest one-per-bus framing; heartbeat moved to lifecycle ownership (`HeartbeatController` + `ProcessLifecycleOwner`); JWT custom-claims contract for RLS; `devices.activation_state` rename; operator-status enum with suspension-at-journey-end; `journey_summaries_pending` outbox; `journey_state` staleness recovery; FR-AT-65 visual flash co-equal with the audio chime; FR-AT-67 GMS detection; FR-AT-04 transient vs terminal recovery; FCM data-only payload; `device_secret` in EncryptedSharedPreferences; manual custom-access-token-hook setup; Sentry crash telemetry.

---

## What This Project Is

A native Android tablet application that displays passenger information on UK buses operating rail replacement services. It runs in kiosk mode, tracks GPS, plays pre-rendered audio announcements from local files, and operates fully offline during journeys. It is paired with a web dashboard (separate repository) where bus operators author routes and manage their fleet.

The tablet does not author routes. It does not synthesise audio. It does not hold a NaPTAN database. Those concerns live on the dashboard or in the server-side render pipeline; the tablet is a display and stop-detection device that consumes pre-rendered artefacts.

The product is legally regulated under the UK Public Service Vehicles (Accessible Information) Regulations 2023. Several rules in this document exist because of those regulations.

---

## Tech Stack

- **Language:** Kotlin (no Java, no cross-platform frameworks)
- **Min SDK:** API 26 (Android 8.0); Target SDK: API 36
- **Architecture:** Clean Architecture, MVVM, single-activity
- **DI:** Hilt
- **Async:** Kotlin Coroutines + StateFlow (no LiveData, no RxJava)
- **Local DB:** Android Room (offline-first — the app reads only from Room during operation)
- **Cloud DB:** Supabase (PostgreSQL) — accessed via Supabase Kotlin SDK
- **Auth:** Supabase Auth (anonymous user per device, created during pairing). JWT carries `operator_id` as a custom claim, stamped by a server-side custom-access-token hook (see Setup Notes and architectural Rule 16).
- **Push:** Firebase Cloud Messaging — data-only payload `{ type, operator_id, trigger }`. The push never carries content; it is a sync trigger only.
- **GPS:** FusedLocationProviderClient in a foreground service
- **Audio:** Android `MediaPlayer` for pre-rendered **WAV files (LINEAR16 PCM, 24 kHz mono)**. Route-specific files are downloaded from Supabase Storage at version-keyed paths (`route-audio/{operator_id}/{route_id}/{route_version}/...`) into `filesDir/audio/`. Fixed announcement files are bundled in the APK under `res/raw/`. **No on-device TTS is used. No MP3 anywhere in the pipeline** — see Rule 9 and the "Audio is LINEAR16/WAV end-to-end" gotcha.
- **Kiosk:** Screen pinning (Lock Task Mode Level 1) for the initial release. Lock Task Mode Level 2 (Device Owner mode) is deferred; the architecture is open to it without redesign but it is not in scope to build. See PRD §FR-AT-46.
- **Background work:** WorkManager for periodic sync fallback and best-effort background heartbeat
- **Heartbeat:** `HeartbeatController` driven by `ProcessLifecycleOwner` (lifecycle-based, not GPS-service-owned). See Rule 15.
- **Crash telemetry:** Sentry (Android SDK). Initialised in `Application.onCreate`; uncaught exceptions and ANRs reported; PII stripped. See PRD §NFR-R-07.

---

## Reference Documents — DO NOT READ

The `/docs` folder in this repo contains four reference documents:

- `/docs/PRD.md` — product requirements
- `/docs/Data-Architecture.md` — database schemas and sync algorithm
- `/docs/Compliance-Mapping-Matrix.md` — regulatory compliance evidence
- `/docs/WORKFLOW.md` — how the project is built

**Do not read these files in normal task work.** They are for the project's architect, not for you. They are also **frozen** — they were finalised at planning time and do not change during the build. Never edit them.

If you think you need information from those documents to complete a task, ask the user instead. The architect will either include the relevant excerpt in the prompt or add a rule to this CLAUDE.md.

The reasons:
- The PRD describes the entire product. Reading it tempts you to build features adjacent to what was asked. Stay focused on the current task.
- Context window is finite. These documents are large.
- If a rule from those documents matters for your work, the architect has already distilled it into the prompt or this file.

---

## Available MCP Tools

Three MCP servers are configured at the user scope and available in every session:

- **Context7** — fetches current official documentation for libraries on demand. Use it before writing code that uses a library where the API may have changed since training (notably Hilt, Room, the Supabase Kotlin SDK, WorkManager, AndroidX libraries generally). A quick Context7 lookup is cheaper than debugging a deprecated API.

- **GitHub** — read-only by convention. You can use it to read repo state, list issues, read PR content. You do **not** push to GitHub, you do **not** open PRs, you do **not** modify repo settings — these are manual operations by the system administrator regardless of what the MCP technically permits.

- **Supabase** — runs in `--read-only` mode by default. You can read schema, query tables, list Edge Functions, inspect storage. You **cannot** run migrations, modify data, or deploy Edge Functions unless the system administrator has explicitly reconfigured the MCP for the current task (the architect's prompt will say so). After any write task, the MCP returns to read-only — do not assume write access persists between prompts.

A fourth MCP, **Playwright**, will be installed before Stage 1 dashboard verification. It is not yet available; do not attempt to invoke it.

When in doubt about whether an MCP-driven operation is authorised for the current task, ask before running it.

---

## Project Structure

The target structure is below. Create directories as needed; do not pre-create empty directories.

    app/src/main/java/com/pds/application/
    ├── data/                  # Room entities, DAOs, repositories, Supabase remote sources
    │   ├── local/             # Room database, entities, DAOs, type converters
    │   ├── remote/            # Supabase client, Edge Function calls, FCM
    │   ├── repository/        # Repository implementations (single source of truth)
    │   └── sync/              # SyncManager, SyncWorker, sync use cases, audio downloader
    ├── domain/                # Use cases, domain models, repository interfaces
    │   ├── model/             # Domain models (Route, Stop, JourneyState)
    │   ├── repository/        # Repository interfaces
    │   └── usecase/           # Use cases (StartJourney, AdvanceStop, etc.)
    ├── presentation/          # UI layer — Activities, Fragments, ViewModels
    │   ├── pairing/           # First-run pairing flow (incl. GMS check, FR-AT-67)
    │   ├── routelist/         # Route list / selection
    │   ├── journey/           # Active journey screen (passenger view + tube-map)
    │   ├── driver/            # Driver control panel overlay
    │   ├── admin/             # Admin menu (PIN-protected)
    │   ├── accountstatus/     # Account Status Screen (pending/suspended; FR-AT-60)
    │   └── common/            # Shared UI components and the custom TubeMapView
    ├── service/               # Foreground GPS service, audio manager, kiosk controller, HeartbeatController
    ├── telemetry/             # Sentry init + PII scrubbing
    ├── di/                    # Hilt modules
    └── util/                  # Extensions, constants, helpers

    app/src/main/res/raw/      # Bundled fixed-announcement audio (see Data-Arch §6.3)
    ├── alert_chime.wav
    ├── silent_keepalive.wav
    ├── termination.wav
    ├── hail_and_ride_start.wav
    ├── hail_and_ride_end.wav
    ├── diversion_start.wav
    └── diversion_end.wav

    # Runtime-resolved audio (not in the source tree — created in app's internal files dir):
    # {context.filesDir}/audio/{operator_id}/{route_id}/{route_version}/
    #     ├── route_announcement.wav
    #     ├── stop_0.wav
    #     ├── stop_1.wav
    #     └── ...
    # route_version = routes.updated_at as epoch millis. Exactly one version per route at a
    # time; older versions are removed on successful download of a new version. All audio is
    # LINEAR16 PCM, 24 kHz mono — see Rule 9.

    docs/                      # Frozen reference documents (DO NOT READ)
    ├── PRD.md
    ├── Data-Architecture.md
    ├── Compliance-Mapping-Matrix.md
    └── WORKFLOW.md

---

## Build, Run, and Test Commands

    ./gradlew assembleDebug              # Build debug APK
    ./gradlew installDebug               # Build and install on connected device/emulator
    ./gradlew test                       # Run unit tests
    ./gradlew connectedAndroidTest       # Run instrumented tests on device/emulator
    ./gradlew clean                      # Clean build outputs

    adb devices                          # List connected devices
    adb logcat -s PDS                    # Filter logcat to PDS tags
    adb uninstall com.pds.application    # Uninstall (needed after Room schema changes)

You can request builds with these commands. You cannot deploy to the physical tablet directly — that's a human action via Android Studio. After a build succeeds, the user verifies on the tablet.

---

## Setup Notes

These setup steps are easy to miss and have non-obvious failure modes if forgotten.

- **Custom access token hook (Supabase Auth console — manual step).** Server-side hook that stamps `operator_id` into the JWT custom claims for every issued token. This is a manual action in the Supabase Auth dashboard; it does NOT travel in migrations. Without it, every operator-scoped RLS policy quietly returns empty and the tablet appears to have no data. See Data-Architecture §12 setup checklist. If a paired tablet "is suddenly empty," check this first.
- **Sentry DSN.** Configured via `gradle.properties` (or env var) and injected into the build at compile time. Do not commit the DSN to git.
- **Firebase project linked.** `google-services.json` placed in `app/`. FCM data-only push depends on this.
- **Google Play Services on the tablet.** The tablet must be a GMS-certified Android device. FR-AT-67 detects this at first run and surfaces a blocking warning if absent.

---

## Architectural Rules — DO NOT VIOLATE

These are the rules whose violation breaks the architecture. Treat them as inviolable. Bracketed tags categorise each rule for quick scanning; rule numbers are stable references (e.g., "Rule 9").

### 1. [Architecture] Presentation never touches Room directly

ViewModels call use cases. Use cases call repository interfaces. Repositories talk to Room and Supabase. No shortcuts.

This is enforced by package boundaries — `presentation` does not import from `data.local` or `data.remote`. If a ViewModel needs data, the use case provides it.

### 2. [Architecture] Room is the single read source during operation

The UI never reads from Supabase. All data flows: Supabase → sync → Room → repository → use case → ViewModel → UI.

The only direct Supabase access happens inside `data.remote` and `data.sync`. Repositories may write to Supabase as part of sync, but reads always come from Room.

### 3. [Sync] Supabase sync never blocks UI

Sync runs in a background coroutine. If sync fails, the app continues operating from cached Room data. No loading spinners waiting for network. The user sees the route list immediately from Room; sync happens silently in the background.

### 4. [Sync] Stops sync as a unit with their parent route

When downloading: UPSERT route, then replace all local stops for that route in a single transaction.

`route_stops` has no `needs_upload`, `is_deleted`, or `updated_at` columns. Stops travel with their route. Never add per-stop sync metadata.

**Scoped exception for `needs_upload`.** The no-`needs_upload` directive applies to `routes` and `route_stops` only. The one legitimate use of `needs_upload` in the initial release is the **`journey_summaries_pending` Room outbox** (Data-Arch §5.8) — a deliberate, narrow exception that supports the journey-summary upload at journey end (FR-AT-66). If you find yourself wanting `needs_upload` on any other Room table, stop and ask.

### 5. [Stop detection] Strict sequential progression with two-stop look-ahead

GPS detection monitors only the **next expected stop (N)** and the **stop after it (N+1)**. Never scan all stops for the closest match. Never advance to a stop other than N or N+1.

Two-stop look-ahead behaviour: if a GPS fix triggers entry to N+1 before N has fired, auto-advance past N — log N as passed without detection, then fire the N+1 announcement. The `stop_order` sequence from route creation is the sole authority for progression order.

This rule prevents misfires when routes loop back, when stops are close together, or when GPS error briefly places the bus near a later stop. See PRD §FR-AT-13.

### 6. [Stop detection] Per-stop proximity radius and per-candidate GPS accuracy gating

Detection radius is **per stop**, set on the dashboard route builder (default 200m). Read from `route_stops.proximity_radius_meters`; do not hard-code a global radius.

Discard GPS fixes whose `horizontalAccuracyMeters` exceeds the **target stop's** configured radius. A bad fix is worse than no fix — better to wait for the next fix than to misfire on a 500m-accuracy reading against a 200m radius. See PRD §FR-AT-13 and dashboard §FR-WD-08.

### 7. [Time] All timestamps in UTC

Room stores epoch millis as LONG. Supabase stores TIMESTAMPTZ. Conversion to local UK time (GMT/BST) happens only at the UI layer for display.

Never compare timestamps from Room to a `Date.getTime()` from a local-timezone source without explicit UTC conversion.

### 8. [Sync] Server assigns sync timestamps

The `updated_at` column on routes is set by a PostgreSQL BEFORE trigger (`now()`), never by the client. The sync cursor (`last_server_timestamp`) is set from the server transaction's `current_timestamp`, not from the max `updated_at` in the batch.

`routes.updated_at` (expressed as epoch millis) is also the `route_version` segment of the audio Storage path scheme — another reason it must originate server-side.

If you find yourself setting `updated_at` from Kotlin code on an outgoing route, stop and ask. That's almost certainly wrong.

### 9. [Audio] Pre-rendered audio playback only — no on-device TTS

Use Android `MediaPlayer` to play pre-rendered **WAV files (LINEAR16 PCM, 24 kHz mono)**. **Do not use the Android `TextToSpeech` API anywhere in this codebase. No MP3 anywhere in the pipeline** — see also the "Audio is LINEAR16/WAV end-to-end" gotcha.

Two sources of audio:
- **Bundled fixed-announcement files** in APK `res/raw/`: `alert_chime.wav`, `termination.wav`, `hail_and_ride_start.wav`, `hail_and_ride_end.wav`, `diversion_start.wav`, `diversion_end.wav`, `silent_keepalive.wav`. These are updated only by shipping a new APK. They were re-rendered once at planning time using the same Google Cloud TTS voice (`en-GB-Neural2-B`, LINEAR16 output) that produces the route-specific files. Reg 13(4) frequency-range presence (300–3000 Hz) has been **empirically verified** on the bundled set — see Compliance Mapping Matrix and `spike-records/round-3/findings-tts-frequency.md`.
- **Route-specific files** synced from Supabase Storage at version-keyed paths to `{filesDir}/audio/{operator_id}/{route_id}/{route_version}/`: `route_announcement.wav` and `stop_{N}.wav` per stop (where `N` is the stop's `stop_order`). `route_version` is `routes.updated_at` in epoch millis. Exactly one version per route is held on the tablet at a time; older versions are deleted on successful download of a new version. Server-side, the `route-audio` Storage bucket retains the latest **three** versions per route — see dashboard CLAUDE.md and Data-Arch §6.

The chime-then-announcement sequence is atomic — the chime must finish before the announcement file begins, and the announcement file must always follow if the chime plays.

### 10. [Audio + Compliance] Alert chime + visual flash for the four regulated announcement types

The following announcement types are subject to the combined-alert requirement:
- Termination (Reg 8(2))
- Diversion start (Reg 10(2)(b))
- Hail-and-ride start (Reg 11(2)(b))
- Hail-and-ride end (Reg 11(5)(b))

Each MUST be:
1. **Preceded by `alert_chime.wav`** (bundled, under 1 second), AND
2. **Accompanied by a 500ms high-contrast screen flash** (FR-AT-65) fired **simultaneously with the chime**, before the announcement overlay text appears.

Both components are **co-equal** — neither alone satisfies the regulation. This is a **legal requirement**, not a UX choice.

Next-stop and route-and-destination announcements get neither the chime nor the flash.

### 11. [Audio gating] `audio_render_status` is the journey-start gate

A journey cannot start unless **both** of the following hold for the selected route:
1. The Room mirror of `routes.audio_render_status = 'ok'`. If it is `pending` or `failed`, the gate is closed regardless of file presence.
2. All expected audio files exist in `{filesDir}/audio/{operator_id}/{route_id}/{route_version}/`. Expected files are derived from the route's stop count: one `route_announcement.wav` plus one `stop_{N}.wav` per stop.

If either check fails, show the **FR-AT-28 "Audio not ready — syncing"** screen and disable the Start Journey button. The screen MUST also display the driver-facing hint **"Contact your fleet manager to verify route audio status."** on a second line (round-3 addition — silent operation is non-compliant under Reg 12(1)(b) so the gate stays a hard block; the hint at least tells the driver who to ask). Trigger a sync. There is no fallback to on-device TTS. See PRD §FR-AT-28 and Data-Arch §6.4.

### 12. [Audio gating] Tablets with `audio_enabled = false` skip audio downloads

The `devices.audio_enabled` flag is a per-device boolean (dashboard-controlled, see dashboard CLAUDE.md Rule 11). The tablet's local mirror lives in the **Room `device_state` single-row table** (Data-Arch §5.9), refreshed at the top of every sync. The audio downloader, the audio playback engine, and the §7.2 step-8 flip-detection logic all read from `device_state.audio_enabled` — **not** from SharedPreferences and **not** from a per-tablet row in a local `devices` table (there is no local `devices` table). When `audio_enabled = false`:

- The sync skips the audio-download step entirely (no `route_announcement.wav` or `stop_{N}.wav` is fetched).
- The audio manager suppresses all audio playback.
- The visual journey UI is displayed in full — tube-map, overlays, flash alert, everything except audio.

**Honest framing.** Software provides the per-device toggle and the dashboard surfaces a warning when more than one tablet in a fleet has audio enabled. **The system cannot enforce one-per-bus** — it does not know which physical bus a tablet is on. Enabling exactly one tablet per bus is operator responsibility. The dashboard's route-list view and the FR-WD-13 global failed-render indicator are the primary surfaces fleet managers use; the corresponding operator practice (verify `routes.audio_render_status = 'ok'` before next service) is named in PRD §11.2 item 9.

### 13. [UI] 22mm minimum text with calibration

Passenger-facing text must be physically 22mm tall. Do not trust `DisplayMetrics` alone — many budget tablets report inaccurate DPI.

Use the stored `screen_calibration_ppmm` value from SharedPreferences (calculated via bank card calibration in admin settings). If uncalibrated (value -1), fall back to `DisplayMetrics` but display a setup-recommended warning.

This rule applies only to *passenger-facing* text — driver controls and admin screens are not regulated and can use normal text sizes.

### 14. [Sync] Tablet is read-only for routes

Routes are authored exclusively on the web dashboard. The tablet displays routes, runs journeys, but does not create, edit, or delete routes.

If you find yourself implementing a "Create Route" button or a NaPTAN search UI on the tablet, stop. That is dashboard-only functionality.

The `routes` table in Room does NOT have a `needs_upload` column for this reason.

### 15. [Heartbeat] Heartbeat is lifecycle-based, not GPS-service-owned

Implementation: a `HeartbeatController` bound to `ProcessLifecycleOwner`. Two paths:
- **App-foregrounded (reliable):** Any Activity in `RESUMED` state — route browsing, route detail, admin menu, active journey. Ticks every 2 minutes; updates `devices.last_seen_at`.
- **Background/idle (best-effort):** `WorkManager` `PeriodicWorkRequest`. Not guaranteed on hostile-OEM hardware; documented acceptable gap.

**The foreground GPS service is not the heartbeat owner.** The two have independent lifecycles and reliability profiles — don't co-locate heartbeat ticks inside the GPS service. Fleet managers should treat idle tablets as "not in service." See PRD §FR-AT-64.

### 16. [Identity + Auth] JWT custom claims (`operator_id`) are the RLS contract

After pairing, the Supabase JWT carries `operator_id` as a custom claim, stamped by the `custom_access_token_hook` registered in the Supabase Auth console (manual setup — see Setup Notes). RLS policies read `operator_id` from the JWT, never from query parameters.

If the hook is misconfigured or unregistered, **every operator-scoped query silently returns empty**. This is the canonical failure mode to check first when "everything is suddenly empty."

### 17. [Telemetry] Sentry crash reporting is required

The Sentry SDK is initialised in `Application.onCreate`. Uncaught exceptions and ANRs are reported. **PII policy (matches PRD §NFR-R-07 and Data-Arch §11.2):** `device_id` is a permitted diagnostic UUID breadcrumb — it contains no personal information and is the single most useful field for correlating a crash to a real tablet, so it is **deliberately retained**. What is stripped: operator names, operator email addresses, passenger info (which the tablet never has anyway), GPS coordinates and any location data, raw stop-name strings from active journeys, and the contents of FCM payloads. Do not disable Sentry in release builds.

See PRD §NFR-R-07 and Data-Architecture §11.2.

---

## Key Data Decisions

**UUIDs as TEXT in Room.** All Supabase UUIDs are stored as TEXT strings in Room. No type conversion at the boundary.

**Single-row tables (`journey_state`, `sync_metadata`, `device_state`).** Use `@PrimaryKey(autoGenerate = false)` with `id` hardcoded to 1, and `@Insert(onConflict = OnConflictStrategy.REPLACE)` in the DAO. This prevents `SQLiteConstraintException` from race conditions — any insert overwrites the existing row. `device_state` (Data-Arch §5.9, added in round-3 Task 3) holds device-level state the tablet maintains locally between syncs — `audio_enabled` and `last_synced_at` initially, forward-compatible for additional cached device-level fields. It is the canonical local source of `audio_enabled` (see Rule 12 and the related gotcha).

**`pending_deletion` flag on routes.** If a remotely deleted route is currently active in a journey, set `pending_deletion = true` instead of `is_deleted = true`. The route stays usable for the active journey but is hidden from the route list. Cleanup runs when the journey ends.

**No NaPTAN on the tablet.** The tablet holds no NaPTAN database. Stop names, CRS codes, latitudes, and longitudes travel with routes via `route_stops` — copied at dashboard route-creation time. NaPTAN is dashboard-only. See CONTEXT Decision #23.

**Route stops copy NaPTAN data at creation.** Stop name, CRS code, latitude, longitude are copied into `route_stops` rather than referenced by foreign key. This means routes survive NaPTAN database changes — and is now structurally necessary because the tablet has no NaPTAN to reference.

**Operator and device identities and credentials.** After pairing:
- Regular `SharedPreferences` (non-secret): `operator_id`, `device_id`, `android_id`.
- `EncryptedSharedPreferences` (secret): `device_secret` (issued at pairing, used for token recovery via `recover-device`), the Supabase JWT, and the refresh token.

The JWT carries `operator_id` as a custom claim. See architectural Rule 16.

**`devices.activation_state` enum** (renamed from `devices.status` in round-2 Task 3). Values per Data-Arch §2.2. The tablet reads this on pairing-check and recovery; an `inactive` device is blocked from journey start.

**`operators.status` three-state enum** (`pending` / `active` / `suspended`). A `suspended` operator's tablets honour suspension **at journey end, not mid-journey** — an in-flight journey runs to its natural end, then on the next journey-start attempt the app transitions to the Account Status Screen (FR-AT-60). The `pending` and `suspended` states are never conflated; each has distinct UX.

**`segment_type` on `route_stops`** and **`journey_skipped_stops` Room table.** `segment_type` (`scheduled` / `hail_and_ride`) drives H&R announcement and tube-map behaviour. `journey_skipped_stops` records stops the operator pre-marks as skipped on a journey instance (diversion handling). See CONTEXT Decision #16.

**`journey_state` recovery rules.** On app restart, recover `journey_state` only if both: `last_event_at` is within the last 1 hour AND the journey is no older than 8 hours since start. Outside those bounds, discard the state. If recovery lands inside a diversion segment, **replay the diversion-start announcement** (full chime + flash + audio sequence) — and that replay sets `diversion_invoked_at_any_point = true` (see latch entry below).

**`journey_state.diversion_invoked_at_any_point` is a one-way latch.** Round-3 addition (Data-Arch §5.5). Set `true` when the driver triggers **FR-AT-25 Diversion Start** (or when a recovery replays a diversion-start). **Never reset to false mid-journey**, even when the driver triggers FR-AT-26 Diversion End. Reset to the default `false` only when a new journey starts. Consumed by the FR-AT-66 journey-summary upload (`diversion_invoked = journey_state.diversion_invoked_at_any_point` evaluated at journey end) so that a journey which ever involved a diversion is reported as such, regardless of whether the diversion was still active at the natural end.

**`journey_events` lifecycle.** Cleared at **both** app startup and journey start (defensive double-cleanup; round-2 Task 3 fix).

**`journey_summaries_pending` Room outbox.** Written at journey end with anonymous count metrics (no PII, no location). Uploaded to Supabase `journey_summaries` by a `WorkManager` job. This is the only Room table with a legitimate `needs_upload` column (see Rule 4 exception). See PRD §FR-AT-66 and Data-Arch §5.8.

**Two route-level hashes.** Don't conflate them:
- `routes.audio_announcement_hash` (Data-Arch §2.4) — SHA-256 of the route-announcement text. Drives the audio render worker's content-hash differential re-render (skip synthesis when unchanged).
- `routes.stops_content_hash` (Data-Arch §2.4) — SHA-256 of the canonical stop-list serialisation. Drives the dashboard's structural FR-WD-12 divergence detection. Not used by the tablet directly; surface via sync only.

---

## Workflow Rules

### Plan before building

Enter plan mode for any task touching 3+ files or involving architectural decisions. Produce a plan, wait for user confirmation, then execute. Do not start changes during plan mode.

### One feature at a time

Complete, test, and commit each feature before starting the next. Reference the PRD requirement being implemented in the commit message (e.g., "feat: implement FR-AT-19 stylised tube-map view"). The user provides the PRD requirement number in the prompt.

### Verify before marking done

Build the project, run tests, check for compilation errors. Do not declare a feature complete without proving the build passes. The user verifies on the tablet — but you should never hand them code that doesn't even compile.

### Keep it simple

Make every change as simple as possible. Touch minimal code. Find root causes, not temporary fixes. If a fix feels hacky, pause and find the clean solution — or ask.

### Do not refactor opportunistically

If you notice unrelated code that "could be improved," leave it. The current task's scope is defined by the prompt. Drive-by refactoring causes scope drift, makes diffs hard to review, and was a recurring problem in the previous build.

### Commit messages

The user (architect) provides the commit message at the end of each task. Use the format:

    <type>: <short title referencing the PRD requirement>

    <detailed description in 2-4 paragraphs covering:
     - what was changed and why
     - which PRD requirements this implements
     - any notable decisions or trade-offs>

`<type>` is one of: `feat`, `fix`, `refactor`, `docs`, `chore`, `test`.

---

## Known Gotchas

These are concrete problems we hit in the previous build attempt — plus a set of round-2 additions for the v3.8 architecture. Avoid them.

### Room schema mismatches require uninstall

If you change a Room entity or DAO in a way that alters the schema (new column, changed type, renamed table), the app will crash on launch with `IllegalStateException: Room cannot verify the data integrity`. The user must uninstall the previous version (`adb uninstall com.pds.application`) and reinstall fresh.

For changes during development, this is fine. For changes after deployment, a Room migration is required. Do not attempt to "fix" the integrity error by changing the expected hash — write a migration instead. See WORKFLOW §10.3.

### Hilt + WorkManager requires getter form

The Application class implements `Configuration.Provider`. The `workManagerConfiguration` must be a **getter property**, not a field:

    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()

A field form causes a Hilt initialisation race where the worker factory is not yet injected when WorkManager reads the configuration. Symptoms: workers fail to instantiate, or the app crashes with "WorkManager already initialised."

Also disable the default `WorkManagerInitializer` via `androidx.startup` in the manifest:

    <provider
        android:name="androidx.startup.InitializationProvider"
        android:authorities="${applicationId}.androidx-startup"
        android:exported="false"
        tools:node="merge">
        <meta-data
            android:name="androidx.work.WorkManagerInitializer"
            android:value="androidx.startup"
            tools:node="remove" />
    </provider>

### Deferrable self-FK on routes

The `return_route_id` column on the `routes` table is a self-foreign-key. When inserting a route pair (outbound + return) in a batch, both rows reference each other. Without a deferrable constraint, the insert order matters and you get FK violations.

On Supabase, the constraint must be `DEFERRABLE INITIALLY DEFERRED`. On Room, this is handled by the order of inserts within a transaction.

### Operator-scoped queries

Repository interfaces require `operator_id` as a parameter for queries that touch operator-scoped data. The repository implementation does not call "get all routes" without scoping. This is a multi-tenancy footgun: an un-scoped query in production would mix routes across operators if the local DB ever held data from multiple operators (which it shouldn't, but defensive coding matters).

If you find yourself adding a `getAllRoutes()` method, stop. Add `getRoutesForOperator(operatorId: String)` instead.

### Foreground GPS service notification

The foreground service must have a visible notification (FOREGROUND_SERVICE_TYPE_LOCATION). On Android 14+, this notification cannot be dismissed by the user during an active journey. Use a low-importance notification channel so it's quiet but persistent.

The notification belongs to the GPS service alone. **Do not co-locate heartbeat ticks here** — heartbeat is owned by `HeartbeatController` / `ProcessLifecycleOwner` (Rule 15).

### EncryptedSharedPreferences key types

EncryptedSharedPreferences only supports String, Int, Long, Float, Boolean, and Set<String>. To store a UUID, convert to String. To store a JWT or `device_secret` (both String), just store as-is.

### Coroutine scope in repositories

Repositories should not hold their own `CoroutineScope`. They are call-site agnostic — the caller (use case or ViewModel) supplies the scope and dispatcher. If you find yourself injecting a scope into a repository, you're probably doing too much in the repository.

The single exception is the sync subsystem, which legitimately runs on its own scope.

### Time zones in display

When showing a timestamp to the user (e.g., "last synced at 14:32"), convert from epoch millis (UTC) to local UK time at the UI layer using `java.time.ZonedDateTime`. Never use `SimpleDateFormat` without an explicit `TimeZone`. The UK switches between GMT and BST; `ZonedDateTime.now(ZoneId.of("Europe/London"))` handles this correctly.

### Audio path mutation

The `route_version` path component (`routes.updated_at` in epoch millis) changes on every successful re-render of the route's audio. Don't cache absolute audio paths in long-lived state — always resolve through the current `route_version` from the Room mirror of `routes`. A stale cached path will point at a directory that has been deleted as part of version cleanup.

### Audio is LINEAR16/WAV end-to-end — no MP3 anywhere

Round-3 Task 2 switched the audio pipeline from MP3 to **LINEAR16 PCM, 24 kHz mono, WAV** end-to-end. The Google Cloud TTS API call uses `audioEncoding: 'LINEAR16'`; the worker writes `.wav` files to the `route-audio` Storage bucket; the tablet downloads and plays `.wav` files with `MediaPlayer` (which plays WAV natively, no extra dependency). The bundled fixed-announcement files in `res/raw/` are also WAV.

Why: Reg 13(4) requires a specific frequency range (300–3000 Hz) be preserved; MP3 perceptual coding cannot be relied on to preserve it without per-file verification. LINEAR16 is lossless and Reg 13(4) frequency presence has been empirically verified on the bundled set (see `spike-records/round-3/findings-tts-frequency.md`). The cost is file size: WAV files are ~10× larger than MP3 (a 5-second stop announcement is ~240 KB). Storage and bandwidth budgets account for this; do not "optimise" by re-introducing MP3 anywhere in the pipeline.

If you see `.mp3` in a path, a Storage call, or a `MediaPlayer` setDataSource argument, that is a bug.

### Local `audio_enabled` lives in Room `device_state`, not SharedPreferences

The tablet's local mirror of its own `devices.audio_enabled` value lives in the **Room `device_state` single-row table** (Data-Arch §5.9). It is **not** in SharedPreferences and there is **not** a local `devices` Room table. The audio downloader, the audio playback engine, and the §7.2 step-8 flip-detection logic all read from `device_state.audio_enabled`. The sync writes `device_state` at the top of the cycle (step 7) and again at the bottom (step 8 ends with an update of `audio_enabled + last_synced_at`). Don't duplicate this value anywhere else.

### Auto-deregistration is a compound condition

The previous "60 days of silence → auto-deregister" rule is now a **compound condition** (Data-Arch §10, round-3 Task 3): a tablet is auto-deregistered only when `last_seen_at` is older than 60 days **AND** `recover_failure_count >= 5` with `last_recover_failure_at` in the trailing 60-day window. Plain silence (seasonal storage, in-repair tablets, dead batteries) never accumulates failure counts and so retains its `devices` row indefinitely — though it still drops out of the billable count after the 30-day silence threshold. Only actively-rejected tablets (operator-decommissioned, hitting terminal `recover-device` failures repeatedly) trip the compound condition. See `devices.recover_failure_count` and `devices.last_recover_failure_at`.

### Custom access token hook misconfiguration

If RLS-scoped queries unexpectedly return empty for a paired tablet, the most likely cause is that the `custom_access_token_hook` is not registered in the Supabase Auth console. The hook is a **manual** dashboard step (not a migration); it is easy to forget after a project reset or environment switch. Check the hook before debugging RLS policies or Kotlin code.

### GMS detection at first run (FR-AT-67)

On first run (and on every subsequent launch as a cheap check), the app must detect Google Play Services availability via `GoogleApiAvailability.isGooglePlayServicesAvailable()`. If unavailable, surface a clear blocking warning. There is a "continue anyway" override (logged to `journey_events`) for development/testing on non-GMS hardware — production tablets must be GMS-certified. Don't crash; don't proceed silently.

### Token recovery — transient vs terminal (FR-AT-04 and FR-AT-50)

The `recover-device` Edge Function distinguishes two failure classes:
- **Transient** (network, 5xx, 429): retain cached credentials, retry with backoff. The tablet stays in service.
- **Terminal** (404, 401, `activation_state = 'inactive'`, operator status not `active`): wipe local credentials and force re-pair.

**Credential wipe scope (parity across FR-AT-04 terminal recovery and FR-AT-50 admin deregister).** When wiping, remove **all three** secrets from EncryptedSharedPreferences in a single transaction: **JWT, refresh token, AND `device_secret`**. The two paths must wipe the same set — an incomplete wipe that leaves the `device_secret` behind would obstruct re-pairing because the tablet would still hold a recovery credential it cannot use. Round-3 Task 3 codified this parity (resolves round-3 finding 10).

Surface different UI for each transient vs terminal recovery — a transient retry banner vs. a "this device has been deactivated, please re-pair" screen. Do not collapse them into a single "recovery failed" state.

### FCM payload schema

The push is **data-only**: `{ type, operator_id, trigger }`. It never carries content. Its sole purpose is to wake the tablet for a sync. Treat any other payload shape as a bug (server-side or client-side).

### Rate-limit attempts

`pair-device` and `recover-device` are rate-limited server-side via the `rate_limit_attempts` table (per-IP and per-Android-ID windows). Don't retry pairing in a tight loop — you'll lock the tablet out. Surface the rate-limit error to the user; the lockouts are short (10-minute and 15-minute windows) and intentional.

### Journey state staleness recovery

When recovering `journey_state` on app restart, enforce **both** age limits: max 1 hour since `last_event_at` AND max 8 hours since journey start. If recovery succeeds and the current segment is a diversion (`segment_type = 'hail_and_ride'` is NOT what to check here — diversion state is in `journey_state`, not `route_stops`), replay the diversion-start announcement in full (chime + flash + audio) so the passenger sees a coherent state.

---

## What to Do When Stuck

If a task seems impossible, contradictory, or under-specified:

1. Stop. Do not guess.
2. Explain what's blocking you to the user. Be specific about what you'd need to know.
3. Wait for clarification.

Do not:
- Read `/docs/` to fill in gaps (the architect distils what you need)
- Implement a "best guess" version that might not match intent
- Refactor unrelated code "while figuring it out"

---

## What This File Is Not

This file is **not** a specification. It does not describe what the app does. The PRD describes that. This file describes **how to work on the app**.

If you need to know "what does feature X do?", ask the user. If you need to know "how do I implement features in this codebase?", read this file.

---

## Sweep Summary (v3.9 changelog detail)

Round-3 deltas (v3.8 → v3.9) now reflected:
- **Audio format switched MP3 → LINEAR16 WAV end-to-end.** File tree, Rule 9, Rule 10, Rule 11, Rule 12, the audio-path-mutation gotcha, and the new "Audio is LINEAR16/WAV end-to-end" gotcha all use `.wav`. Google Cloud TTS call uses `audioEncoding: 'LINEAR16'`. `MediaPlayer` plays WAV natively.
- **Reg 13(4) frequency-range empirically verified** on the bundled set (see Compliance Mapping Matrix and `spike-records/round-3/findings-tts-frequency.md`); no longer a pre-deployment blocker.
- **Rule 17 Sentry PII rewritten.** `device_id` is a permitted diagnostic UUID breadcrumb (no personal information). Stripped fields: operator names/emails, passenger info, location/GPS, raw stop-name strings from active journeys, FCM payloads. Matches PRD §NFR-R-07 and Data-Arch §11.2.
- **FR-AT-50 wipe scope clarified** to enumerate JWT + refresh token + device_secret in EncryptedSharedPreferences. Parity with FR-AT-04 terminal recovery codified in the gotcha.
- **Compound 60-day auto-deregistration** (silence AND ≥5 failed `recover-device` attempts) replaces the simple 60-day silence rule. Distinguishes seasonal/in-repair silence from active rejection. References `devices.recover_failure_count` and `devices.last_recover_failure_at`.
- **`device_state` Room single-row table** (Data-Arch §5.9) is the canonical local source of `audio_enabled` — referenced in the Single-row tables decision, Rule 12, and the new "Local `audio_enabled` lives in Room `device_state`" gotcha.
- **`journey_state.diversion_invoked_at_any_point` one-way latch** documented in the Key Data Decisions section. Set on diversion-start; never reset mid-journey; reset to default only on a new journey; consumed by FR-AT-66 journey-summary upload.
- **FR-AT-28 "Audio not ready — syncing" screen** now spec'd to include the driver hint "Contact your fleet manager to verify route audio status." Operator-side practice (verify `audio_render_status = 'ok'` before next service) named in Rule 12 with the FR-WD-13 indicator surface.

---

## Sweep Summary (v3.8 changelog detail)

Round-1 deltas (v3.0 → v3.7) now reflected:
- Tablet NaPTAN bundle removed; no first-launch import. Stop data travels with routes.
- Upload-sync rule (old "Rule 8") removed; sync is download-only.
- Kiosk Level 2 (Device Owner) deferred; Level 1 screen pinning is initial release.
- Operator-status three-state enum (`pending` / `active` / `suspended`) replaces `is_approved`.
- Pre-rendered audio architecture documented; on-device TTS explicitly forbidden.
- `device_secret` added to EncryptedSharedPreferences alongside JWT + refresh token.
- JWT custom-claims contract (Rule 16) made explicit.
- Per-stop `proximity_radius_meters` + GPS accuracy gate (Rule 6).
- Two-stop look-ahead progression (Rule 5).
- `segment_type` on `route_stops`; `journey_skipped_stops` Room table.
- Version-keyed Storage paths; `filesDir/audio/{operator_id}/{route_id}/{route_version}/` layout.
- Per-device `audio_enabled` flag with download skip (Rule 12).
- `audio_render_status` journey-start gate (Rule 11).
- FR-AT-65 visual flash made co-equal with the audio chime (Rule 10).

Round-2 deltas (v3.7 → v3.8) now reflected:
- Heartbeat moved to lifecycle ownership: `HeartbeatController` + `ProcessLifecycleOwner` (Rule 15). GPS service is no longer the heartbeat owner.
- Audio pipeline: `pg_boss` job queue + Google Cloud TTS (voice `en-GB-Neural2-B`) + version-keyed paths + render-then-FCM ordering + differential re-render via `audio_announcement_hash`. The deprecated synchronous `render-route-audio` Edge Function is gone; replaced by `enqueue-render-job` (orchestrator) + `audio-render-worker` (queue consumer).
- `audio_enabled` operator-responsibility framing made honest (Rule 12).
- Operator suspension honoured at journey end, not mid-journey.
- Sentry crash telemetry adopted on Android (initialised in `Application.onCreate`).
- Journey summary upload at journey end via `journey_summaries_pending` outbox (FR-AT-66).
- `audio_enabled = false` tablets skip audio downloads entirely (not just playback).
- `stops_content_hash` for structural divergence detection (dashboard side; tablet syncs the column).
- `journey_state` staleness recovery (8h/1h bounds); diversion announcement replay.
- `journey_events` cleanup at both app startup and journey start.
- FR-AT-67 GMS detection at first run.
- FR-AT-04 transient vs terminal recovery classification.
- `devices.status` renamed `devices.activation_state` (CHECK constraint).
- 30-day heartbeat-billable; compound 60-day auto-deregister (silence AND ≥5 failed recover-device attempts — refined in round-3 Task 3, see "Auto-deregistration is a compound condition" gotcha). Seasonal/in-repair tablets retain their `devices` row indefinitely; only actively-rejected tablets are auto-deregistered.
- `rate_limit_attempts` table for `pair-device` / `recover-device`.
- FCM data-only payload schema `{ type, operator_id, trigger }`.
- Custom access token hook registration is a manual Supabase Auth console step.
- Anonymous Supabase Auth user accumulation acknowledged as deferred operational concern.

---

## End of CLAUDE.md
