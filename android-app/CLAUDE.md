# CLAUDE.md
# Passenger Display System (PDS) — Android Application

This file is your reference for working on the PDS Android application. Read it at the start of every session. It contains conventions, architectural rules, and known gotchas. If a rule here conflicts with a prompt, ask before deviating.

---

## What This Project Is

A native Android tablet application that displays passenger information on UK buses operating rail replacement services. It runs in kiosk mode, tracks GPS, plays pre-rendered audio announcements from local files, and operates fully offline during journeys. It is paired with a web dashboard (separate repository) where bus operators author routes and manage their fleet.

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
- **Auth:** Supabase Auth (anonymous user per device, created during pairing)
- **Push:** Firebase Cloud Messaging (triggers sync, never carries data)
- **GPS:** FusedLocationProviderClient in a foreground service
- **Audio:** Android MediaPlayer for pre-rendered MP3 files (route-specific files synced from Supabase Storage; fixed announcement files bundled in APK). No on-device TTS is used.
- **Kiosk:** Lock Task Mode (Level 2 — Device Owner) or screen pinning (Level 1)
- **Background work:** WorkManager for periodic sync fallback

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
    │   └── sync/              # SyncManager, SyncWorker, sync use cases
    ├── domain/                # Use cases, domain models, repository interfaces
    │   ├── model/             # Domain models (Route, Stop, JourneyState)
    │   ├── repository/        # Repository interfaces
    │   └── usecase/           # Use cases (StartJourney, AdvanceStop, etc.)
    ├── presentation/          # UI layer — Activities, Fragments, ViewModels
    │   ├── pairing/           # First-run pairing flow
    │   ├── routelist/         # Route list / selection
    │   ├── journey/           # Active journey screen (passenger view + tube-map)
    │   ├── driver/            # Driver control panel overlay
    │   ├── admin/             # Admin menu (PIN-protected)
    │   └── common/            # Shared UI components and the custom TubeMapView
    ├── service/               # Foreground GPS service, audio manager, kiosk controller
    ├── di/                    # Hilt modules
    └── util/                  # Extensions, constants, helpers

    app/src/main/assets/
    ├── naptan_stations.json.gz  # ~25–30MB compressed UK NaPTAN data
    ├── alert_chime.mp3          # Regulatory alert tone (under 1 second)
    └── silent_keepalive.mp3     # Bluetooth speaker keepalive (1 second, inaudible)

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

## Architectural Rules — DO NOT VIOLATE

These are the rules whose violation breaks the architecture. Treat them as inviolable.

### 1. Presentation never touches Room directly

ViewModels call use cases. Use cases call repository interfaces. Repositories talk to Room and Supabase. No shortcuts.

This is enforced by package boundaries — `presentation` does not import from `data.local` or `data.remote`. If a ViewModel needs data, the use case provides it.

### 2. Room is the single read source during operation

The UI never reads from Supabase. All data flows: Supabase → sync → Room → repository → use case → ViewModel → UI.

The only direct Supabase access happens inside `data.remote` and `data.sync`. Repositories may write to Supabase as part of sync, but reads always come from Room.

### 3. Supabase sync never blocks UI

Sync runs in a background coroutine. If sync fails, the app continues operating from cached Room data. No loading spinners waiting for network. The user sees the route list immediately from Room; sync happens silently in the background.

### 4. Stops sync with their parent route, not independently

When uploading a route (future feature): UPSERT route, then delete-and-reinsert all stops for that route via the `replace_route_with_stops` RPC.

When downloading: UPSERT route, then replace all local stops for that route.

`route_stops` has no `needs_upload`, `is_deleted`, or `updated_at` columns. Stops travel with their route. Never add per-stop sync metadata.

### 5. Strict sequential stop progression

GPS detection only monitors proximity to the NEXT expected stop in the route's stop_order sequence. Never scan all stops for the closest match. Never advance to a stop other than the next one.

This rule exists to prevent misfires when routes loop back, when stops are close together, or when GPS error briefly places the bus near a later stop. The stop_order sequence from route creation is the sole authority for progression order.

### 6. All timestamps in UTC

Room stores epoch millis as LONG. Supabase stores TIMESTAMPTZ. Conversion to local UK time (GMT/BST) happens only at the UI layer for display.

Never compare timestamps from Room to a `Date.getTime()` from a local-timezone source without explicit UTC conversion.

### 7. Server assigns sync timestamps

The `updated_at` column on routes is set by a PostgreSQL BEFORE trigger (`now()`), never by the client. The sync cursor (`last_server_timestamp`) is set from the server transaction's `current_timestamp`, not from the max `updated_at` in the batch.

If you find yourself setting `updated_at` from Kotlin code on an outgoing route, stop and ask. That's almost certainly wrong.

### 8. Upload before download, abort if upload fails

Sync sequence: check account status → upload local changes → download remote changes. If upload fails, do not proceed to download — the download could overwrite un-synced local edits.

In the current release, tablets have no local edits to upload (route authoring is dashboard-only), so the upload phase is effectively a no-op. The sequence is preserved for architectural correctness and future-proofing.

### 9. Alert chime before certain announcements

The following announcements MUST be preceded by the bundled `alert_chime.mp3` (under 1 second):
- Termination (Reg 8(2))
- Diversion start (Reg 10(2)(b))
- Hail-and-ride start (Reg 11(2)(b))
- Hail-and-ride end (Reg 11(5)(b))

This is a **legal requirement**, not a UX choice. Next-stop and route-and-destination announcements do NOT get an alert chime.

When implementing audio playback, ensure the chime-then-announcement sequence is atomic — the chime must finish before the announcement file begins, and the announcement file must always follow if the chime plays. Both the chime and all announcement audio are pre-rendered MP3 files; do not use the Android TextToSpeech API anywhere in this codebase.

### 10. 22mm minimum text with calibration

Passenger-facing text must be physically 22mm tall. Do not trust `DisplayMetrics` alone — many budget tablets report inaccurate DPI.

Use the stored `screen_calibration_ppmm` value from SharedPreferences (calculated via bank card calibration in admin settings). If uncalibrated (value -1), fall back to `DisplayMetrics` but display a setup-recommended warning.

This rule applies only to *passenger-facing* text — driver controls and admin screens are not regulated and can use normal text sizes.

### 11. Tablet is read-only for routes

Routes are authored exclusively on the web dashboard. The tablet displays routes, runs journeys, but does not create, edit, or delete routes.

If you find yourself implementing a "Create Route" button or a NaPTAN search UI on the tablet, stop. That is dashboard-only functionality.

The `routes` table in Room does NOT have a `needs_upload` column for this reason.

---

## Key Data Decisions

**UUIDs as TEXT in Room.** All Supabase UUIDs are stored as TEXT strings in Room. No type conversion at the boundary.

**Single-row tables (`journey_state`, `sync_metadata`).** Use `@PrimaryKey(autoGenerate = false)` with `id` hardcoded to 1, and `@Insert(onConflict = OnConflictStrategy.REPLACE)` in the DAO. This prevents `SQLiteConstraintException` from race conditions — any insert overwrites the existing row.

**`pending_deletion` flag on routes.** If a remotely deleted route is currently active in a journey, set `pending_deletion = true` instead of `is_deleted = true`. The route stays usable for the active journey but is hidden from the route list. Cleanup runs when the journey ends.

**NaPTAN data is bundled.** Loaded from `assets/naptan_stations.json.gz` into Room on first launch and on APK update. Not synced from Supabase. The initial load takes 30–60 seconds; display a progress UI.

**Route stops copy NaPTAN data at creation.** Stop name, CRS code, latitude, longitude are copied into `route_stops` rather than referenced by foreign key. This means routes survive NaPTAN database changes.

**Operator and device identities come from SharedPreferences.** After pairing, `operator_id`, `device_id`, and `android_id` live in regular SharedPreferences. The Supabase JWT and refresh token live in EncryptedSharedPreferences.

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

These are concrete problems we hit in the previous build attempt. Avoid them.

### Room schema mismatches require uninstall

If you change a Room entity or DAO in a way that alters the schema (new column, changed type, renamed table), the app will crash on launch with `IllegalStateException: Room cannot verify the data integrity`. The user must uninstall the previous version (`adb uninstall com.pds.application`) and reinstall fresh.

For changes during development, this is fine. For changes after deployment, a Room migration is required. Do not attempt to "fix" the integrity error by changing the expected hash — write a migration instead.

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

### EncryptedSharedPreferences key types

EncryptedSharedPreferences only supports String, Int, Long, Float, Boolean, and Set<String>. To store a UUID, convert to String. To store a JWT (which is always String), just store as-is.

### Coroutine scope in repositories

Repositories should not hold their own `CoroutineScope`. They are call-site agnostic — the caller (use case or ViewModel) supplies the scope and dispatcher. If you find yourself injecting a scope into a repository, you're probably doing too much in the repository.

The single exception is the sync subsystem, which legitimately runs on its own scope.

### Time zones in display

When showing a timestamp to the user (e.g., "last synced at 14:32"), convert from epoch millis (UTC) to local UK time at the UI layer using `java.time.ZonedDateTime`. Never use `SimpleDateFormat` without an explicit `TimeZone`. The UK switches between GMT and BST; `ZonedDateTime.now(ZoneId.of("Europe/London"))` handles this correctly.

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

## End of CLAUDE.md
