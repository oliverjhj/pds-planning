# Product Requirements Document (PRD)
# Passenger Display System (PDS)

**Version:** 3.7
**Last Updated:** May 2026
**Status:** Pre-Development

## Changelog

### v3.7 (May 2026)
Post-adversarial-review re-planning pass, item 7 of 7 (compliance, WORKFLOW, smaller items, and campaign close-out).
- FR-AT-40: Added non-occlusion constraint — driver panel must not cover the next-stop name or route name; panel occupies lower portion of screen only.
- FR-AT-65 (new): Visual Alert — 500ms high-contrast screen flash fired simultaneously with the audio chime, before announcement overlay text appears; same four trigger types as the audio chime.
- FR-WD-12: Added return-route divergence detection (`last_synced_with_return` timestamp; dashboard warning on edit with [Re-generate] option).
- FR-AT-35: Added verification step after initial calibration (50mm reference bar + user confirmation; recalibrate if user disagrees).
- §5.2: Multi-tablet audio model specified — one `audio_enabled` flag per device; first-paired tablet defaults audio-on, subsequent default audio-off; configurable from dashboard fleet view.

### v3.6 (May 2026)
Post-adversarial-review re-planning pass, item 6 of 7 (scope cuts: Kiosk Level 2 deferred, tablet NaPTAN bundle removed, upload-sync scaffolding removed).
- FR-AT-46: Level 1 soft kiosk is the only kiosk mode in the initial release. Level 2 (Device Owner) deferred to Could Have; architecture accommodates it without redesign.
- FR-AT-55: Removed "NaPTAN data" from offline-operation description (tablet holds no NaPTAN).
- FR-AT-57: Removed upload step from sync sequence. Sync is download-only. Removed "abort on upload failure" language. Renumbered steps.
- NFR-P-04: Annotated as achievable from first launch — NaPTAN import no longer exists.
- NFR-S-07: Removed Level 2 kiosk specificity (kiosk Level 2 is deferred).
- §7.2: Removed "On the tablet" NaPTAN paragraph; replaced with accurate description (tablet holds no NaPTAN).
- §7.3: Removed Tablet NaPTAN update bullet. Revised closing explanation.
- §9 MoSCoW: "Kiosk mode (Levels 1 and 2)" → "Kiosk mode (Level 1 soft kiosk)"; Level 2 added to Could Have.
- §13 Risks: Removed "NaPTAN dataset size impacts tablet first-launch UX" row (import gone).

### v3.5 (May 2026)
Post-adversarial-review re-planning pass, item 5 of 7 (sync triggers, FCM, and fleet status).
- FR-WD-15: Redefined "online" — based on `last_seen_at` within 5 minutes (maintained by heartbeat), not sync recency.
- FR-AT-56: Reframed all three sync triggers as core; FCM described as the responsive trigger enabling sub-5-minute propagation.
- FR-AT-62: Narrowed to sync-only `last_seen_at` update; heartbeat is covered by new FR-AT-64.
- FR-AT-64 (new): Heartbeat Mechanism — 2-minute periodic `last_seen_at` update independent of route sync, two-path implementation.
- §9 MoSCoW: FCM moved from Should Have to Must Have (added to both Android Tablet and Backend sections).
- §12: 5-minute route-propagation metric annotated with explicit FCM dependency.

### v3.4 (May 2026)
Post-adversarial-review re-planning pass, item 4 of 7 (auth, device identity, and security).
- FR-WD-01: `is_approved = false` → `status = 'pending'`.
- FR-WD-02: Added email retry note and pending-operators SQL reference (see Data Architecture §3.1).
- FR-WD-03: `is_approved = true` → `status = 'active'`.
- FR-WD-04: Updated tablet pre-approval message — `pending` shows "Account pending approval", not "Account Suspended."
- FR-WD-05: Complete rewrite. `suspended` ≠ `pending`. Distinct UX messaging for each state on both surfaces. `is_approved = false` replaced with `status = 'suspended'`.
- FR-AT-02: `pair-device` now generates a 256-bit device secret; plaintext returned once to tablet; stored in EncryptedSharedPreferences. JWT custom claims noted (see Data Architecture §3.4a).
- FR-AT-04: `recover-device` now requires both Android ID and device secret.
- FR-AT-57: Sync step 1 updated — status check discriminates `pending` vs `suspended` with distinct messages.
- FR-AT-60: Complete rewrite. Screen title changed to "Account Status Screen"; discriminates by status with distinct messages for `pending` and `suspended`.
- NFR-S-06: Rewritten to describe the device secret model accurately. Android ID is an identifier, not a credential.
- NFR-S-07 (new): Stolen-tablet threat model. Names the threat, states the mitigation (FR-WD-17 device deactivation), acknowledges the accepted gap.

### v3.3 (May 2026)
Post-adversarial-review re-planning pass, item 3 of 7 (audio architecture — pre-rendered files).
- §1.1: Updated product description to reflect pre-rendered audio (not TTS).
- FR-WD-19 (new, renumbered): Route Audio Rendering — dashboard calls `render-route-audio` Edge Function after route save.
- FR-AT-22, FR-AT-23: Added source note (pre-rendered route-specific audio files).
- FR-AT-24: Updated to reference bundled `termination.mp3`; removed TTS reference.
- FR-AT-25: Rewritten. Audio portion is now a single fixed pre-rendered phrase; visual display carries stop-specific detail. Dynamic per-stop audio naming removed.
- FR-AT-26: Added source note (bundled `hail_and_ride_start/end.mp3`).
- FR-AT-28: Complete rewrite. "TTS Engine" → "Audio Playback Engine". Describes pre-rendered MP3 file model, route-specific vs. bundled files, journey-start gating. On-device TTS removed entirely.
- FR-AT-33: Updated "TTS content" → "pre-rendered announcement audio file".
- FR-AT-63: Replaced `TTS failures` event with `AUDIO_FILE_MISSING` and `AUDIO_PLAYBACK_ERROR`.
- NFR-R-03: Rewritten for audio file failure (not TTS failure).
- NFR-A-02: Updated wording to reflect server-side rendering.
- §6.1: Updated offline audio bullet.
- §9 MoSCoW: Pre-rendered audio moved from Could Have to Must Have; TTS item removed; audio rendering added to Backend Must Have.
- §11.2: Removed Android TextToSpeech API dependency.
- §13 Risks: Updated TTS quality risk row.

### v3.2 (May 2026)
Post-adversarial-review re-planning pass, item 2 of 7 (hail-and-ride and diversion data model).
- FR-WD-08: Added `segment_type` authoring to the route builder (per-stop "Scheduled" / "Hail and ride" selector).
- FR-AT-13: Added H&R section handling (silent traversal, automatic boundary announcements) and diversion skip handling (auto-advance through `journey_skipped_stops`).
- FR-AT-19: Updated tube-map rendering to show H&R sections as dashed lines with diamond nodes, and skipped stops with strikethrough.
- FR-AT-25: Rewritten with dynamic announcement content that names skipped stops; fallback to generic text if no stops selected.
- FR-AT-26: Rewritten — H&R announcements now automatically triggered at segment boundaries; manual buttons retained as fallback.
- FR-AT-41: Updated diversion and H&R button descriptions to reflect new automatic/fallback model.
- FR-AT-42: Added diversion stop selector for marking stop ranges as skipped.

### v3.1 (May 2026)
Post-adversarial-review re-planning pass, item 1 of 7 (stop detection).
- FR-WD-08: Added per-stop proximity radius authoring to the route builder.
- FR-AT-11: Added non-stop-0 journey start detection. App checks bus position after GPS service starts and prompts driver to confirm starting from a mid-route stop if applicable.
- FR-AT-13: Major rewrite. Global proximity radius replaced with per-stop `proximity_radius_meters` from `route_stops`. Added GPS accuracy gate (fix discarded if accuracy worse than target stop's radius). Added two-stop look-ahead (monitors N and N+1; auto-advances past N if N+1 is entered first, logging N as passed without detection).
- FR-AT-14: Clarified departure detection uses per-stop radius.
- FR-AT-49: Removed "Proximity radius adjustment" from admin menu (radius is now per-stop, set on dashboard).
- NFR-P-01: Wording fix ("the stop's proximity radius").
- MoSCoW Must Have: Updated proximity radius bullet to per-stop formulation.

---

## 1. Product Overview

### 1.1 What This Is

A two-surface SaaS product providing legally compliant audio and visual passenger information on UK buses operating rail replacement services. The product consists of:

1. **A web dashboard** (Next.js) where bus operators register, manage routes, and pair their devices.
2. **An Android tablet application** that runs on each bus, displays journey progress, announces stops via pre-rendered audio files, and meets the requirements of the Public Service Vehicles (Accessible Information) Regulations 2023.

The two surfaces share a single Supabase backend. They ship together. Operators cannot use one without the other.

### 1.2 The Problem

UK disability legislation requires buses to display route and stop information visually and announce it audibly. Legacy B2B providers charge approximately £5,000 per vehicle for hardware/software bundles. Smaller operators — particularly those running rail replacement services — need a dramatically cheaper solution that they can install on their own off-the-shelf hardware.

### 1.3 The Solution

A SaaS software product that runs on operator-supplied Android tablets paired with a Bluetooth speaker. The software handles route management (via the dashboard), GPS-triggered announcements (on the tablet), compliant visual displays, and offline-first journey operation. By unbundling hardware from software, the product undercuts incumbents on total cost of ownership while keeping the regulatory compliance burden squarely on the software side.

### 1.4 Business Model

Monthly subscription, charged per active tablet. Initial pricing target: £30–50 per tablet per month. Billing is handled manually (invoice) for the initial customer set; automated billing via Stripe is planned for a later release once pricing is validated and the customer count justifies the integration cost. A device is considered active if it has synced within the last 30 days.

### 1.5 Target Market

UK bus operators running rail replacement services. Initial pilot: a single bus company. Expansion to broader UK bus operators and route types after validation.

---

## 2. User Personas

### 2.1 The Driver (Tablet User)

- Operates the bus and interacts with the tablet at the start and end of each journey, plus occasional mid-journey actions
- Selects a pre-built route from a list — does not create or edit routes
- Triggers manual announcements when needed (diversions, hail-and-ride, termination)
- Is not technical; the tablet interface must be obvious and require no training beyond a brief walkthrough
- Wears gloves in cold weather; touch targets must be large

### 2.2 The Fleet Manager (Dashboard User)

- Works at the bus company's office or depot
- Logs into the web dashboard to create routes, manage devices, and administer the operator account
- Searches NaPTAN data to build routes (railway stations and bus stops)
- Generates pairing codes when registering new tablets
- Sees a fleet view of all registered devices and their online/offline status
- May manage 5–50 vehicles

### 2.3 The Passenger

- Never interacts with the system directly
- Views the passenger display on the tablet showing route progress and the upcoming stop
- Hears audio announcements for the next stop and other regulatory events
- May be visually impaired, deaf or hard of hearing, have cognitive impairments, or use a wheelchair in a rearward-facing space
- Needs information clear, timely, and consistent between audio and visual formats

### 2.4 The System Administrator (You)

- Operates the SaaS product
- Approves new operator signups via SQL queries against Supabase
- Invoices customers manually
- Maintains the dashboard, the Android app, and the Supabase project
- Receives email notification when a new operator signs up

---

## 3. Functional Requirements — Web Dashboard

### 3.1 Operator Account Management

**FR-WD-01: Operator Self-Signup**
The dashboard shall allow new operators to sign up by providing an email address, a password, and a company name. Supabase Auth handles password hashing, email verification, and password recovery. On signup, an `operators` row is created with `status = 'pending'`. The operator can log in immediately but the dashboard shows a "Pending approval" state until the system administrator approves the account.

**FR-WD-02: System Administrator Notification**
On every successful operator signup, the system shall email the system administrator's address with the new operator's company name, email, and signup timestamp. This is the trigger for the manual approval workflow. The email is the notification trigger, but the database is the authoritative pending queue — if the email fails, the system retries once after 10 minutes (see Data Architecture §3.1). After the retry, no further automatic notification is sent; the system administrator can query pending operators directly via SQL at any time (see Data Architecture §3.1 for the query).

**FR-WD-03: Operator Approval (Administrator Action)**
The system administrator approves an operator by setting `status = 'active'` on the operator's row in Supabase. This is performed via a direct SQL query in the initial release; no admin UI is built. Once approved, the operator's dashboard transitions from the pending state to full functionality on next page load.

**FR-WD-04: Pending-Approval Restrictions**
While `status = 'pending'`, the dashboard restricts the operator to viewing the "Account pending approval" message and editing their account profile. They cannot create routes, generate pairing codes, or perform any action whose effect would propagate to a tablet. Tablets paired before approval (a state that should not normally occur — FR-WD-14 requires `status = 'active'` to generate a pairing code) show "Account pending approval — contact your administrator." They do not show "Account Suspended" — that message is reserved for the `suspended` state.

**FR-WD-05: Account Suspension**
The system administrator can suspend an approved operator by setting `status = 'suspended'` on the operator's `operators` row (e.g., for non-payment). Suspension is distinct from the `pending` state: a suspended account was previously active; a pending account has never been approved.

**Dashboard behaviour:** A suspended operator's dashboard shows "Account suspended" and restricts functionality to the same degree as the pending state (no route creation, no pairing code generation, no actions that propagate to tablets). The messaging differs: "Account suspended" for `suspended` vs. "Account pending approval" for `pending`.

**Tablet behaviour:** On the suspended operator's next sync, tablets display "Account Suspended — please contact your bus company administrator" and disable all journey functionality. This message is not shown for `pending` accounts; those show "Account pending approval — contact your administrator."

Suspension is reversible: setting `status = 'active'` restores full functionality on next dashboard page load and next tablet sync.

**FR-WD-06: Single Login Per Operator**
Each operator account has exactly one Supabase Auth user associated with it. There are no team or sub-user accounts. If multiple people at a bus company need access, they share the login.

**FR-WD-07: Account Profile Management**
Operators can update their company name, email address (with verification), and password through the dashboard. Account deletion is not self-service in the initial release; operators contact the system administrator to be removed.

### 3.2 Route Management

**FR-WD-08: Route Creation**
The dashboard provides a route builder where operators name a route, set a route number and direction label, and add ordered stops by searching NaPTAN data. Stops are added by clicking from the search results into an ordered list, then can be reordered by drag-and-drop or up/down controls.

For each stop in the route, the builder displays a proximity radius field pre-filled with 200 metres. Operators can adjust this value per stop to suit local conditions — larger for motorway-services stops where the bus decelerates over a long approach distance, smaller for dense town-centre stops where adjacent stops are close together. The value is saved in `route_stops.proximity_radius_meters` and synced to tablets as part of the route.

For each stop, the builder also shows a segment type selector, defaulting to "Scheduled stop." The operator can switch individual stops to "Hail and ride" to mark them as part of a hail-and-ride section. Consecutive stops marked as hail and ride form a hail-and-ride section; the dashboard visually groups them as a section in the stop list. The value is saved in `route_stops.segment_type` and synced to tablets as part of the route.

**FR-WD-09: NaPTAN Search**
The dashboard provides full-text search across all UK NaPTAN entries (bus stops and railway stations, ~400,000 records). Search results display the stop name, NaPTAN identifier, locality, and (for railway stations) the 3-letter CRS code, to disambiguate similarly named stops. Search is performed against the `naptan_stations` table in Supabase using Postgres full-text search.

**FR-WD-10: Route Metadata**
Each route shall have:
- A display name (e.g., "Newcastle to Carlisle")
- An optional route number or short identifier
- A direction label (e.g., "Outbound" or "Return")
- An optional link to a return route (see FR-WD-12)

**FR-WD-11: Route Editing and Deletion**
Operators can edit existing routes (rename, reorder stops, add/remove stops, change metadata) and delete routes. Edits and deletions are scoped to the operator's own routes (RLS-enforced). Deletion is soft (sets `is_deleted = true`) so the deletion can propagate to all paired tablets via sync.

**FR-WD-12: Return Route Generation**
When saving a route, the dashboard offers a "Generate return route" action that creates a separate route entity with the same stops in reverse order and the direction label flipped. The two routes are linked via `return_route_id` so each can navigate to its counterpart. The two routes can be edited independently after generation.

**Return-route divergence detection:** Once a return route has been generated and either route is subsequently edited, the routes may diverge (different stops, different order). To surface this, each route carries a `last_synced_with_return` timestamp (see Data Architecture §2.4). When a return route is generated, both routes have `last_synced_with_return` set to the moment of generation. On the dashboard, when an operator saves an edit to a route that has a `return_route_id` AND the route's `updated_at` is newer than its `last_synced_with_return`, the dashboard displays a warning: *"This route has a linked return route that may now be divergent. [Re-generate return route] [Keep existing return]."* Choosing [Re-generate] replaces the linked return route's stop list with the current route's stops in reverse order, resets `last_synced_with_return` on both routes, and triggers audio re-rendering for the return route. Choosing [Keep existing return] dismisses the warning without further action. This is a warning-and-regenerate mechanism, not a diff UI. The operator is responsible for reviewing the linked route if they choose to keep it.

**FR-WD-13: Route List View**
The dashboard provides a list of all routes belonging to the operator, with name, route number, direction, stop count, and last-modified timestamp. Routes can be filtered or searched by name. Soft-deleted routes are hidden from this view.

### 3.3 Device Management

**FR-WD-14: Device Pairing — Code Generation**
The dashboard provides an "Add Device" action that calls a Supabase Edge Function to generate a single-use 6-digit pairing code. The code is stored in a `device_pairing_codes` table with the operator's ID, the code, an expiry timestamp 10 minutes in the future, and `used = false`. The dashboard displays the code prominently with a countdown timer until expiry.

**FR-WD-15: Device Fleet View**
The dashboard shows a list of all devices registered to the operator, displaying:
- Human-readable device name (operator-assigned, e.g., "Bus #42")
- Online/offline status (online if `last_seen_at` is within the last 5 minutes — maintained by the heartbeat mechanism, FR-AT-64)
- Last-seen timestamp
- Device's currently active route, if any (read from the device's last sync state)
- Device registration date
- Android device identifier (for support purposes)

**FR-WD-16: Device Naming**
After a device is paired, the dashboard allows the fleet manager to assign a human-readable name. Default name on first pair is "New Device" with a sequence number. The device name is operator-only metadata; the tablet itself does not display it.

**FR-WD-17: Device Deactivation**
The dashboard provides a "Deactivate device" action that sets the device's status to `inactive` in Supabase, revoking its JWT on next refresh attempt. A deactivated device must re-pair to function again. This is used when a tablet is lost, stolen, or decommissioned, and is reflected in billing (deactivated devices do not count toward the monthly billable count).

### 3.4 Dashboard Hosting and Infrastructure

**FR-WD-18: Hosting**
The dashboard is deployed on Vercel using Next.js. The free tier is sufficient for the initial customer set. The dashboard's domain is owned by the system administrator.

**FR-WD-19: Secure Connection**
All dashboard traffic uses HTTPS exclusively. No mixed-content or insecure connections are permitted.

**FR-WD-20: Route Audio Rendering**
When an operator saves a route (creates or modifies), the dashboard calls the `render-route-audio` Edge Function asynchronously after the route is persisted. The Edge Function renders route-specific audio files using a server-side TTS API with a single configured voice:
- Route announcement: "This bus is the [Route Name] service to [Final Stop]."
- Per-stop next-stop announcement for each stop N: "Next stop: [Stop Name]."

Rendered files are stored in Supabase Storage, keyed by operator ID, route ID, and stop order (see Data Architecture §2.7). The dashboard does not block on rendering; the route appears immediately in the operator's route list. Tablets download audio files on their next sync after rendering completes.

Fixed announcement texts (termination, hail-and-ride start/end, diversion start/end) are not rendered at route-save time — they are rendered once and bundled in the APK, using the same TTS voice as the route-specific files, to ensure a consistent audio experience across all announcement types.

---

## 4. Functional Requirements — Android Tablet Application

### 4.1 First-Run Setup and Device Pairing

**FR-AT-01: First-Run Setup Screen**
On first launch, before any other functionality is available, the app presents a setup screen requiring the user to enter the 6-digit pairing code generated on the web dashboard. No other input is required.

**FR-AT-02: Pairing Code Validation and Device Registration**
On submission of the pairing code, the app calls a Supabase Edge Function (`pair-device`) passing the code and the device's Android ID. The Edge Function validates that the code exists, is unused, and has not expired. On success, it creates a `devices` row, marks the pairing code as used, and returns a long-lived device JWT with `operator_id` and `device_id` as custom claims (stamped automatically at token issuance by the Supabase Auth custom access token hook — see Data Architecture §3.4a). In addition, `pair-device` generates a cryptographically random 256-bit device secret. The plaintext secret is returned in the pairing response alongside the JWT. The tablet stores this secret in EncryptedSharedPreferences alongside the JWT and refresh token. The server stores only a SHA-256 hash of the secret (`devices.device_secret_hash`); the plaintext is never retained or logged server-side after the response is sent. The device secret is set once at pairing and remains constant for the lifetime of the device registration. On failure (invalid, expired, or already-used code), the function returns an error and the app displays a clear failure message ("Pairing code invalid or expired. Generate a new code from your dashboard.").

**FR-AT-03: JWT Storage**
The device JWT is stored in EncryptedSharedPreferences. The accompanying refresh token is stored alongside. The device's pairing details (operator ID, device ID, Android ID) are stored in regular SharedPreferences for diagnostic purposes.

**FR-AT-04: JWT Refresh and Recovery**
The Supabase client library refreshes the JWT automatically when it approaches expiry. If the refresh token has itself expired (e.g., the device sat unused for many months), the app does NOT prompt the user to re-pair. Instead, the app silently calls the `recover-device` Edge Function, passing both the stored Android ID and the stored device secret. The Android ID identifies the device row; the device secret authenticates the request (the server hashes the presented secret and compares against the stored `device_secret_hash`). On success, a fresh JWT is returned and the device continues without re-pairing. The device secret is stable — it does not change across token recoveries. This means that once a device is paired, it stays paired until explicitly deactivated from the dashboard. If `recover-device` fails (device deactivated, secret mismatch, row missing), the app shows a clear message and returns to the first-run setup screen for re-pairing.

**FR-AT-05: Setup Persistence**
Once a device is paired, the setup screen is never shown again. The pairing details persist across app restarts and device reboots. The only way to return to the setup screen is via the admin "Deregister Device" action (FR-KM-03), which requires the admin PIN.

### 4.2 Route Browsing (No Creation)

**FR-AT-06: Route List Screen**
The driver sees a list of all routes for their operator, sorted by recent usage (most recently used first), with name, route number, direction, and stop count. The list is loaded from the local Room database. There is no route creation, editing, or deletion on the tablet — these are dashboard-only operations.

**FR-AT-07: Route Filtering**
The driver can filter the route list by typing into a search box. Filtering operates on route name and route number.

**FR-AT-08: Route Detail View**
Tapping a route shows a detail view with the full stop list (in order), route metadata, and a "Start Journey" button. The detail view is read-only.

**FR-AT-09: Sync Status Indicator**
The route list screen displays the time of the last successful sync and a visual indicator of sync state (synced, syncing, sync failed, never synced). The indicator is unobtrusive but visible.

**FR-AT-10: Empty Route List**
If no routes are available (e.g., the operator hasn't created any yet, or the device has just been paired and hasn't synced), the route list shows a friendly empty state: "No routes available. Create routes on your dashboard." plus the dashboard URL for reference.

### 4.3 Journey Operation

**FR-AT-11: Journey Start**
The driver selects a route and taps "Start Journey." The app shows a confirmation dialog with route name, stop count, and first/last stop names. On confirmation, the app:
- Initialises the active journey (writes to `journey_state`)
- Starts the foreground GPS service
- Takes an initial GPS fix (with accuracy estimate)

**Non-stop-0 start detection:** After starting the GPS service, the app checks the bus's current GPS position against the route's stop sequence. It evaluates stops with index 1 through `stopCount − 2` inclusive — every stop except stop 0 and the final stop. For each candidate stop, the check uses that stop's `proximity_radius_meters` value, and the fix is only counted if its accuracy estimate is within that value. If the bus is within the proximity radius of stop N (1 ≤ N ≤ stopCount − 2), the app presents a prompt: *"Bus appears to be near [Stop N name]. Start journey from this stop?"* with buttons *"Yes, start from [Stop N name]"* and *"No, start from stop 0."* On confirmation, `journey_state.current_stop_index` is set to N and the app begins monitoring from there. On decline, `current_stop_index` is set to 0. If no stop in the scan range matches (fix is too inaccurate, bus is not near any mid-route stop, or is only near stop 0 or the final stop), the app silently starts from stop 0. If two stops are simultaneously within their respective radii, the lower index takes precedence.

After the start index is resolved, the app:
- Switches to the journey screen
- Plays the route-and-destination announcement (FR-AT-22)
- Begins monitoring proximity to the first unannounced stop

**FR-AT-12: Journey Screen Layout**
The journey screen has two regions:
- **Top two-thirds (passenger information area):** Route name, direction, large next-stop name (regulation 22mm minimum), and an announcement overlay that appears when announcements fire. This area is what passengers read.
- **Bottom one-third (route progress visualisation):** A horizontal stylised "tube map" view showing all stops on the route as circular nodes connected by a line, with the bus's current position rendered as a moving dot, the next stop highlighted, completed stops dimmed, and upcoming stops in neutral styling. See section 4.4.

The journey screen is the primary passenger-facing display. Driver controls are accessed via a tap-zone overlay (see FR-AT-29).

**FR-AT-13: Automatic Stop Detection**
The app continuously monitors GPS position via FusedLocationProviderClient running in a foreground service. On each position fix:

**GPS accuracy gate:** Every fix from FusedLocationProviderClient includes an accuracy estimate (the 68%-confidence radius, in metres). Before running any detection logic, compare this estimate to the current target stop's `proximity_radius_meters` value read from Room. If the fix accuracy is worse than (greater than) that value, discard the fix — no announcement fires and `journey_state` is not advanced. This prevents a bad fix from misfiring in an urban canyon or at journey start.

**Two-stop look-ahead:** The app monitors proximity to both the next expected stop (N) and the stop immediately after it (N+1), each evaluated against its own `proximity_radius_meters` value.

- **Normal path — stop N entered:** If the fix passes the accuracy gate for stop N and the bus is within stop N's proximity radius, the app fires the next-stop announcement for N, advances `journey_state.current_stop_index` to N+1, refreshes the visual display, and logs a `STOP_ANNOUNCED` event (trigger = `GPS`).
- **Look-ahead path — stop N+1 entered without N:** If the fix passes the accuracy gate for stop N+1 and the bus is within stop N+1's proximity radius, but stop N was never registered, the app logs a `STOP_PASSED_WITHOUT_DETECTION` event for N (trigger = `GPS_INFERRED`) and a `STOP_ANNOUNCED` event for N+1 (trigger = `GPS_INFERRED`), announces N+1, advances `current_stop_index` to N+2, and marks stop N as passed (dimmed) on the tube-map view. The driver is not required to take any action.

**Critical design rule — strict sequential progression with two-stop tolerance:** The app monitors only stops N and N+1. It never scans all stops for the closest match and never automatically skips more than one stop. The two-stop window is a missed-detection tolerance mechanism for dense routes — not a "closest stop wins" strategy. Stop order defined at route creation is the sole authority for progression. A fix near stop N+2 or beyond is ignored until N+1 has been processed.

**Hail-and-ride section handling:** The GPS state machine reads `segment_type` from Room for each stop it evaluates. When `current_stop_index` advances to a stop with `segment_type = 'hail_and_ride'` and the preceding stop was `scheduled`, the state machine fires the hail-and-ride section start announcement (FR-AT-26) and then continues advancing through hail-and-ride stops silently — no "Next stop: [name]" announcement fires for any hail-and-ride stop. When the state machine next encounters a `scheduled` stop after a run of hail-and-ride stops, it fires the hail-and-ride section end announcement (FR-AT-26) immediately before the "Next stop" announcement for that scheduled stop. GPS proximity is still checked for hail-and-ride stops (using their `proximity_radius_meters` value and the accuracy gate) to allow the bus's position to be tracked on the tube-map; advancing through an H&R stop is silent but not skipped in the index.

If the entire route consists of hail-and-ride stops (all `segment_type = 'hail_and_ride'`), the hail-and-ride start announcement fires at journey start (FR-AT-11), before the first stop monitoring begins.

**Diversion skip handling:** When `current_stop_index` advances to a stop whose index is in `journey_skipped_stops` (FR-AT-42), the state machine immediately increments the index without GPS proximity detection and without firing any announcement. A `STOP_SKIPPED` event is logged. This repeats until the state machine reaches a non-skipped stop, which resumes normal detection. The two-stop look-ahead still applies to the first non-skipped stop after the skipped range.

**FR-AT-14: Stop Departure Detection**
After a stop is announced, the app waits for the bus to exit that stop's `proximity_radius_meters` radius before re-arming detection for the next stop. This prevents repeated announcements while the bus is stationary at a stop.

**FR-AT-15: Manual Stop Advance**
The driver can manually advance to the next stop or go back to the previous stop via the driver control panel, overriding GPS detection. Tapping any stop in the route's stop list jumps the journey position directly to that stop. This handles GPS failures, missed stops, or unscheduled detours.

**FR-AT-16: Journey Completion**
When the bus arrives at the final stop, the app plays the termination announcement (FR-AT-23) and transitions to the journey-end screen after a 10-second delay. The journey-end screen offers:
- "Start Return Journey" if a return route is linked
- "Select New Route"
- "End Session" (returns to idle screen)

**FR-AT-17: Journey Auto-Timeout**
To prevent battery drain if the driver forgets to end a journey, the app automatically terminates the journey and returns to the idle screen if the bus remains stationary at the final stop for 15 minutes (configurable in admin settings).

**FR-AT-18: Journey State Recovery**
If the app is killed during an active journey (crash, OS kill, unexpected reboot), the journey state is recovered on next launch. The journey_state row indicates the active route and current stop index. The app resumes the journey from this state without driver intervention.

### 4.4 Stylised Route Progress View

**FR-AT-19: Tube-Map Rendering**
The bottom third of the journey screen renders a custom Android View showing the route as a horizontal stylised map:
- A horizontal line spans the width of the screen
- Each stop is a circular node positioned along the line, evenly spaced
- Hail-and-ride stops are rendered as smaller diamond nodes; the line segment spanning a hail-and-ride section is dashed rather than solid, visually distinguishing it from the scheduled portions of the route
- Skipped/diverted stops (whose index is in `journey_skipped_stops`) are rendered with a strikethrough marker, visually distinct from both completed (dimmed) and upcoming (neutral) stops
- The current bus position is a distinct dot that moves smoothly along the line as the bus progresses
- The next unannounced stop is highlighted with a brighter colour and a label above the node
- Completed stops are dimmed (low opacity)
- Upcoming scheduled stops use neutral styling
- Stop names are visible above their nodes when there's room; if the route has many stops, only the next-stop name is rendered to avoid clutter

The view is rendered using Canvas drawing on a custom View subclass. No mapping APIs, no third-party libraries.

**FR-AT-20: Update Frequency**
The view re-renders only on stop progression events (next-stop announcement fires) and on a low-frequency tick (every 30 seconds) to update the inter-stop bus position. It does not re-render on every GPS update, to conserve battery. Position between stops is interpolated based on time elapsed since the previous stop and an estimated travel time.

**FR-AT-21: Long Routes**
For routes with more than 10 stops, the view auto-scales: the line shows all stops but smaller, with only the current/adjacent stop names labelled. The current position remains roughly centred. The full stop list is always available via the driver control panel for reference.

### 4.5 Audio Announcements

**FR-AT-22: Route and Destination Announcement**
At the start of a journey and after each stop departure, the app plays the pre-rendered route announcement audio file (`route_announcement.mp3` for the active route, synced from Supabase Storage). The file contains: "This bus is the [Route Name] service to [Final Stop Name]."

**FR-AT-23: Next Stop Announcement**
When entering proximity of the next stop (or when the driver manually advances), the app plays the pre-rendered next-stop audio file for that stop (`stop_{N}.mp3` for the active route, synced from Supabase Storage). The file contains: "Next stop: [Stop Name]."

**FR-AT-24: Termination Announcement**
At the final stop, the app plays the alert chime (Reg 8(2)) followed by the bundled pre-rendered termination audio file (`termination.mp3`), which contains: "This service terminates here. All change, please."

**FR-AT-25: Diversion Announcement**
The driver triggers a diversion announcement via the driver control panel (FR-AT-41).

**Audio:** The app plays the alert chime (Reg 10(2)(b)) followed by the bundled pre-rendered diversion announcement file (`diversion_start.mp3`), which contains: "This service is on diversion. Please check the display for affected stops." This audio is a fixed phrase, consistent across all tablets.

**Visual:** The passenger display simultaneously shows which specific stops are being skipped, via the tube-map strikethrough rendering (FR-AT-19). Stop names from `route_stops.stop_name` for each index in `journey_skipped_stops` are shown on-screen with a strikethrough marker. The visual display carries the stop-specific detail that the audio does not name (see Compliance Mapping Matrix Reg 10(1) for the compliance argument).

The driver must first open the diversion stop selector (FR-AT-42) to mark upcoming stops as skipped before triggering the announcement; the audio and visual updates are then triggered together. If no stops are marked, the audio still plays but the visual shows no strikethrough stops.

**Diversion end:** When the driver triggers "Diversion end," the app plays the alert chime followed by the bundled pre-rendered `diversion_end.mp3`: "This service has resumed its normal route." All entries in `journey_skipped_stops` are cleared and the tube-map strikethrough markers are removed.

**FR-AT-26: Hail-and-Ride Announcements**
Hail-and-ride section start and end announcements are triggered automatically by the GPS state machine when it crosses a segment boundary in the route data (FR-AT-13). All announcements are played from bundled pre-rendered audio files:

- **Section start:** When the GPS state machine announces the last scheduled stop before a hail-and-ride section (i.e., when `segment_type` of the NEXT stop is `hail_and_ride`), it automatically plays: the alert chime (Reg 11(2)(b)) followed by `hail_and_ride_start.mp3` — "You are now entering a hail and ride section. Please signal the driver if you wish to alight."
- **Section end:** When the GPS state machine detects approach to the first scheduled stop after a hail-and-ride section (i.e., when the current stop's `segment_type` is `scheduled` and the preceding stop's was `hail_and_ride`), it automatically plays: the alert chime (Reg 11(5)(b)) followed by `hail_and_ride_end.mp3` — "You are now leaving the hail and ride section." The `stop_{N}.mp3` announcement for that scheduled stop follows immediately.
- **Entire-route hail and ride:** If all stops in the route have `segment_type = 'hail_and_ride'`, the section start announcement fires at journey start (FR-AT-11) before GPS monitoring begins.

The driver control panel (FR-AT-41) retains manual H&R start and end buttons as a fallback for GPS failure or correction. If the driver manually triggers H&R start or end, the state machine behaves as if the corresponding automatic boundary was crossed: H&R start silences subsequent stop announcements; H&R end re-enables them. The manual buttons are available at all times during an active journey.

**FR-AT-27: Alert Chime**
The alert chime is a bundled audio file (under 1 second) played immediately before termination, diversion, and hail-and-ride announcements. The same chime is used for all four to provide a consistent passenger cue. Next-stop and route-and-destination announcements do NOT have an alert chime.

**FR-AT-28: Audio Playback Engine**
All tablet audio announcements are played from pre-rendered MP3 files. The tablet does not use the Android TextToSpeech API; on-device TTS is not installed, configured, or used.

There are two categories of audio file:

**Route-specific files** (stored in Supabase Storage; synced to the tablet during route sync — see Data Architecture §2.7 and §6.4):
- `route_announcement.mp3` — "This bus is the [Route Name] service to [Final Stop]."
- `stop_{N}.mp3` for each stop N — "Next stop: [Stop Name]."

**Bundled files** (packaged in the APK; identical for all operators — see Data Architecture §6.3):
- `termination.mp3` — "This service terminates here. All change, please."
- `hail_and_ride_start.mp3` — "You are now entering a hail and ride section. Please signal the driver if you wish to alight."
- `hail_and_ride_end.mp3` — "You are now leaving the hail and ride section."
- `diversion_start.mp3` — "This service is on diversion. Please check the display for affected stops."
- `diversion_end.mp3` — "This service has resumed its normal route."

All files are rendered using a single consistent server-side voice at route-save time (FR-WD-20), ensuring that every tablet in the fleet produces identical audio for any given announcement regardless of tablet model or manufacturer.

**Journey-start gating:** Before enabling the "Start Journey" button for a route, the app verifies that all expected audio files for that route are present in local storage. If any are missing (route newly synced, server-side rendering still in progress, or interrupted download), the route is shown with an "Audio not ready — syncing" indicator and cannot be started. This is a clear error state; there is no fallback to on-device TTS.

**FR-AT-29: Audio Output Routing**
Audio is routed to the connected Bluetooth speaker or wired audio output. If no external device is connected, audio plays through the tablet's built-in speaker. Audio focus is requested via AudioFocusRequest with AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK.

**FR-AT-30: Audio Device Disconnect Handling**
If the connected Bluetooth speaker disconnects mid-journey, the app:
- Falls back automatically to the tablet's built-in speaker
- Continues all announcements without interruption
- Displays a persistent driver-side warning ("External speaker disconnected — using tablet speaker") that remains until the speaker reconnects or the driver dismisses it

**FR-AT-31: Bluetooth Keep-Alive**
During an active journey, the app plays a silent (inaudible) 1-second audio file every 10 minutes to prevent Bluetooth speakers from auto-powering-down due to inactivity. The keep-alive is bundled audio, not TTS.

**FR-AT-32: Audio-Visual Consistency**
Every audio announcement is accompanied by a matching visual update on the passenger display. Audio and visual content are generated from the same underlying data, ensuring consistency by design (Reg 12(1)(b)).

**FR-AT-33: Audio Concurrency**
If a manual announcement (e.g., diversion) is triggered while an automatic announcement (e.g., next-stop) is playing, the manual announcement interrupts and replaces the current audio. The alert chime for the new announcement plays first, then the pre-rendered announcement audio file.

### 4.6 Visual Display Requirements

**FR-AT-34: 22mm Minimum Character Height**
All passenger-facing text (next-stop, route name, announcement overlays) is rendered at a minimum physical height of 22mm. The app initially calculates font size from `DisplayMetrics`, but because budget tablets often report inaccurate DPI, a calibration step in admin settings allows manual correction (see FR-AT-35).

**FR-AT-35: Screen Calibration**
The admin settings menu includes a "Calibrate screen" action that displays a rectangle on screen and asks the user to adjust a slider until it matches a standard bank card (85.6mm × 53.98mm). The resulting pixels-per-mm ratio is stored in SharedPreferences as `screen_calibration_ppmm` and used for all subsequent 22mm rendering. Calibration is performed once during device setup and can be redone via admin settings.

**Verification step:** After the initial slider calibration, the app displays a second screen showing a solid horizontal bar explicitly labelled as 50mm wide. The user is asked to measure this bar against their bank card (85.6mm reference edge) and confirm: "Does the bar appear to be approximately 50mm?" with [Yes, looks correct] and [No, recalibrate] buttons. If the user selects [No, recalibrate], the calibration flow restarts from the slider screen. This step catches gross calibration errors (e.g. a 10% slider error that would silently produce 19.8mm text against the 22mm legal requirement). It is not bulletproof against deliberate miscalibration, but it catches honest operator errors.

**FR-AT-36: Mixed Case Only**
No passenger-facing text is rendered in all-caps. Mixed case is used throughout, as required by Reg 14(5)(a).

**FR-AT-37: High Contrast**
The passenger display uses high-contrast colour schemes (light text on dark background, or vice versa) suitable for varying lighting from bright sunlight to night.

**FR-AT-38: Announcement Overlay**
During an announcement, a large overlay appears on the passenger display showing the announcement text. The overlay remains visible for 8 seconds (configurable) before fading back to the standard journey view.

**FR-AT-39: Idle Screen**
When no journey is active, the tablet shows a clean idle screen with the app name. Operator branding is not supported in the initial release.

**FR-AT-65: Visual Alert**
Regulations 8(2), 10(2)(b), 11(2)(b), and 11(5)(b) require an alert "immediately preceding" four announcement types. The audio chime (FR-AT-27) satisfies the audible element. The visual element is a distinct 500ms screen flash fired simultaneously with the audio chime, before the announcement overlay text (FR-AT-38) appears.

Specification:
- **Duration:** 500ms.
- **Colour:** High-contrast inverted scheme relative to the resting display (e.g. a white-on-dark display flashes to a bright amber or white full-screen fill) — distinct from both the normal display and the announcement overlay.
- **Timing:** Fires at the same instant as the chime starts playing. The announcement overlay text (FR-AT-38) appears immediately after the 500ms flash, while the chime audio is still playing.
- **Trigger:** Same four announcement types as the audio chime: route termination, diversion start, hail-and-ride start, hail-and-ride end. Does NOT fire for next-stop or route-and-destination announcements.
- **Relationship to overlay:** The flash is a separate, brief visual event. The announcement overlay (FR-AT-38) follows. They do not compete — the flash completes before overlay text is shown.

### 4.7 Driver Controls

**FR-AT-40: Driver Control Panel**
A designated tap zone on the screen (e.g., a small icon or corner area) opens the driver control panel as an overlay. The panel auto-dismisses after 30 seconds of inactivity, returning the screen to the passenger view.

**Non-occlusion constraint (compliance):** The driver control panel must not occlude the next-stop name or the route name. These are the legally-required passenger-facing elements under Regulations 7 and 9. In the physical layout, the panel opens as an overlay occupying **the lower portion of the screen** (the route-progress / tube-map area), while the passenger information area (top two-thirds: route name, direction label, large next-stop name) remains fully visible above the panel at all times. The panel never extends into the top two-thirds. This constraint applies in both portrait and landscape orientations.

**FR-AT-41: Manual Announcement Triggers**
The control panel includes buttons for:
- **Diversion start:** Opens the diversion stop selector (FR-AT-42), generates the dynamic diversion announcement (FR-AT-25) from the selected stops, and fires it with the alert chime. While a diversion is active, a "Diversion active — tap to update skipped stops" indicator replaces this button.
- **Diversion end:** Fires "This service has resumed its normal route." (with alert chime) and clears all entries in `journey_skipped_stops`. Restores the "Diversion start" button.
- **Hail-and-ride start (manual fallback):** Fires the H&R start announcement with alert chime, regardless of segment boundaries. Use when GPS fails to detect the boundary or when correction is needed.
- **Hail-and-ride end (manual fallback):** Fires the H&R end announcement with alert chime, regardless of segment boundaries.
- **Repeat last announcement**

**FR-AT-42: Stop Navigation Controls**
The panel includes:
- "Next stop" button (manual advance)
- "Previous stop" button (manual rewind)
- A scrollable list of all stops with the ability to tap any stop to jump the journey position to it

**Diversion stop selector:** When the driver initiates a diversion (FR-AT-41), a stop selector overlay appears showing all upcoming stops from `current_stop_index` to the final stop. The driver taps individual stops to mark or unmark them as skipped. The selected stops are written to `journey_skipped_stops`. The driver can also re-open the selector while a diversion is active (via the FR-AT-41 "Diversion active" indicator) to update the skipped set. Marked stops are shown with a visual indicator (e.g., a strikethrough) in the selector list and in the tube-map view.

**FR-AT-43: End Journey**
A clearly labelled "End Journey" button triggers the termination announcement and returns the app to the route selection screen.

**FR-AT-44: Volume Control**
The control panel includes a volume slider. The app intercepts hardware volume buttons to prevent accidental muting. The minimum volume is floored at 30% of maximum (configurable) so announcements remain audible. An "Emergency Mute" toggle is available for safety situations and bypasses the floor.

**FR-AT-45: Driver Panel Restrictions**
The control panel does not expose route editing, account settings, or any operation that affects multiple journeys. Settings that affect more than the current journey are accessed via the admin menu (PIN-protected).

### 4.8 Kiosk Mode and Device Management

**FR-AT-46: Kiosk Lock**
The app runs in a locked-down mode preventing exit, home-screen access, or switching to other apps. The initial release implements **Level 1 (soft kiosk)**: the app registers as the device's default home launcher and uses Android's screen pinning. This is suitable for the pilot operator and cooperative deployments where fleet managers set up tablets themselves.

Kiosk Level 2 (Device Owner mode / Android Lock Task Mode) is deferred to a future release — see §9 Could Have. The app architecture is designed to accommodate Level 2 without redesign; only the provisioning differs. Level 2 provisioning is not implemented or documented in the initial release.

**FR-AT-47: Boot on Startup**
The app launches automatically on device boot, with no user interaction required. Tablets must be configured with no lock-screen PIN, password, or pattern (use "None" or "Swipe") because Android's lock screen would otherwise block launch.

**FR-AT-48: Admin PIN Unlock**
A configurable 6+ digit admin PIN allows authorised personnel to exit kiosk mode for maintenance, app updates, or troubleshooting. Entering the PIN unpins the screen (Level 1 soft kiosk) and gives access to the Android home screen, settings, and package installer. Returning to the app re-engages kiosk mode automatically.

**FR-AT-49: Admin Menu**
The admin menu, accessed via PIN, includes:
- Screen calibration
- Volume floor adjustment
- Auto-timeout adjustment
- Journey event log viewer
- Force-sync action
- Deregister device

**FR-AT-50: Deregister Device**
The "Deregister device" admin action wipes all local route data, removes the device JWT, and returns the app to the first-run setup screen. The corresponding `devices` row in Supabase is set to `inactive`. This is the only way to re-pair a device with a different operator account.

**FR-AT-51: Battery Optimisation Bypass**
The app requests `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` to prevent Android's battery management from killing the foreground GPS service.

**FR-AT-52: Foreground Service**
GPS tracking runs in a persistent foreground service (FOREGROUND_SERVICE_TYPE_LOCATION) with a visible notification, ensuring it survives OS attempts to free memory.

**FR-AT-53: Screen Wakelock**
During an active journey, the app holds `FLAG_KEEP_SCREEN_ON` to prevent the screen from sleeping. When idle (no journey), the tablet's normal screen timeout applies.

**FR-AT-54: App Updates**
App updates are delivered by sideloading APK files. The system administrator builds the updated APK, transfers it to the tablet (USB, cloud link, or email), and installs via the admin-unlocked package installer. Google Play Store distribution is a future consideration.

### 4.9 Sync and Offline Operation

**FR-AT-55: Offline-First Operation**
The app operates fully offline during journeys. All route data and stop data are read from the local Room database. Network connectivity is required only for initial pairing, periodic sync of route changes, and JWT refresh.

**FR-AT-56: Sync Triggers**
Route sync is initiated by three core triggers:
- ConnectivityManager detecting offline-to-online transition — handles routes that changed while the tablet was offline.
- Firebase Cloud Messaging (FCM) push notification when an operator's routes change — the responsive trigger that delivers sub-5-minute propagation to an already-online tablet. FCM is a Must-Have feature (§9).
- Periodic check every 30 minutes if continuously online — safety-net backstop for the rare case where FCM and the connectivity-change trigger are both missed.

Note: the heartbeat (FR-AT-64) is a separate mechanism and does not trigger a route sync.

**FR-AT-57: Sync Sequence**
Each sync runs in this order:
1. Check operator account status. If `status = 'pending'`, lock the UI to an "Account pending approval — contact your administrator" screen and abort. If `status = 'suspended'`, lock the UI to an "Account Suspended — please contact your bus company administrator" screen and abort.
2. Download remote changes (routes added/modified/deleted by the operator on the dashboard).
3. Update last-sync state in sync_metadata.

**FR-AT-58: Sync Failure Handling**
If sync fails, the app continues to operate from the local Room database. The sync status indicator shows "failed" until the next successful sync. The app does not retry aggressively on failure (to avoid battery drain); it waits for the next regular trigger.

**FR-AT-59: Conflict Resolution**
Last-write-wins, with the server-assigned `updated_at` timestamp as authority. Conflicts are extremely rare given dashboard-only authoring (no concurrent multi-device edits) but the architecture handles them correctly via the sync algorithm specified in the Data Architecture document.

**FR-AT-60: Account Status Screen**
If a sync detects the operator's account is not in `active` status, the app displays a status-specific screen and disables all journey functionality:
- `status = 'pending'`: displays "Account pending approval — contact your administrator."
- `status = 'suspended'`: displays "Account Suspended — please contact your bus company administrator."

The screen rechecks status on each sync. When the operator's status returns to `active`, the app returns to normal operation without requiring re-pairing.

These two states must never show the same message. A brand-new operator whose tablet was paired before account approval (an unusual state, since FR-WD-14 rejects pairing code generation for non-active operators) sees the pending message, not the suspended message.

**FR-AT-61: Route Deletion During Active Use**
If a route is deleted on the dashboard while a tablet is mid-journey on that route, the active journey is unaffected (operates from local cached state). On next sync after the journey ends, the deletion is applied locally. If the driver attempts to start a journey on a now-deleted route, the app displays "This route is no longer available" and returns to the route selection screen. The app never crashes due to remotely deleted routes.

**FR-AT-62: Device Last-Seen Update on Sync**
On every successful sync, the device updates its `last_seen_at` in Supabase. This supplements the heartbeat (FR-AT-64), which is the primary mechanism for keeping `last_seen_at` current. The `last_seen_at` timestamp drives the dashboard's online/offline display (FR-WD-15) and the active-device count for billing.

**FR-AT-63: Journey Event Logging**
The app silently logs key events to the local Room database for diagnostic purposes:
- Journey started (route ID, timestamp)
- Stop announced (stop name, trigger method [GPS/manual], timestamp)
- GPS signal lost / regained
- Audio device connected / disconnected
- Audio file missing (route audio not yet synced when journey start attempted)
- Audio playback errors (file corrupt or MediaPlayer failure mid-journey)
- Journey ended
- Significant clock drift events (when sync detects local clock differs from server by >5 minutes)
- App errors

Logs are stored locally only (not synced). They are accessible via the admin menu for troubleshooting. Logs older than 30 days are auto-deleted on app startup.

**FR-AT-64: Heartbeat Mechanism**
The app sends a lightweight heartbeat to Supabase every 2 minutes whenever it has connectivity. The heartbeat performs a single operation: updating `devices.last_seen_at` to the current timestamp. It is independent of route sync and runs during active journeys as well as when the app is idle.

Implementation:
- **During active journeys:** A 2-minute `Handler.postDelayed` loop runs inside the foreground GPS service. The foreground service is protected from OEM battery-optimisation kills, ensuring the heartbeat is reliable during the critical in-service window.
- **When idle/backgrounded:** A WorkManager `PeriodicWorkRequest` with a 2-minute interval handles heartbeats when no journey is active.

Heartbeat failures are silent. There is no UI indication of a missed heartbeat, no retry, and no impact on journey operation. The next successful heartbeat updates the timestamp.

The heartbeat interval (2 minutes) and online threshold (5 minutes, see FR-WD-15) are chosen so that a healthy tablet comfortably maintains "online" status: 2-minute heartbeat + 3-minute network-hiccup margin = 5-minute threshold.

---

## 5. Physical Device Placement and Driver Interaction

This section explains how tablets are physically positioned on the bus and why the architecture exists as it does. It is informational rather than a list of requirements.

### 5.1 The Tablet (Driver-Facing and Passenger-Facing)

The tablet is mounted near the driver's seat, within arm's reach — typically on or near the dashboard, or on a flexible arm mount attached to the driver's partition. The driver interacts with it while seated and never needs to leave their seat.

The tablet serves a dual purpose. For most of the journey, it displays the passenger-facing view — route progress, next stop, announcements. This means the tablet itself acts as a passenger display for seats near the front. When the driver needs to interact (selecting a route, triggering a manual announcement), they tap a designated zone on the screen to open the driver control panel as a temporary overlay. After interaction, the overlay auto-dismisses back to the passenger view.

Audio announcements play from a Bluetooth speaker (or wired speaker) connected to the tablet. The speaker should be positioned centrally in the bus for maximum coverage. Placement is the operator's responsibility.

### 5.2 Multi-Tablet Configurations (Operator-Driven)

For larger buses where a single front-mounted tablet cannot meet the 51% sightline requirement, operators may deploy multiple tablets running the same software. In the initial release, each tablet runs independently — no shared journey state, no primary-secondary GPS linking. Each tablet is paired separately and runs its own GPS service; because they're tracking the same physical bus, their journey progress will be approximately synchronised.

**Audio designation:** In any multi-tablet deployment, audio is the responsibility of exactly one designated tablet per bus — the **primary tablet**. Other tablets run in **display-only mode**: they show all visual passenger information (route name, next stop, announcement overlays, tube-map progress) but suppress all audio output, including announcements and the alert chime. Running multiple tablets with audio enabled simultaneously would produce out-of-sync audio — the same announcement playing slightly offset from two speakers — which is non-compliant with Reg 12(1)(b)'s consistency requirement.

The designation is controlled by a per-device `audio_enabled` flag in the `devices` table, configurable from the dashboard fleet view. The fleet manager sees an "Audio" toggle next to each device in the fleet list and can enable it on exactly one device per bus. **Default at pairing time:** the first tablet paired to an operator account has `audio_enabled = true`; every subsequent tablet defaults to `audio_enabled = false`. Operators deploying a single tablet require no configuration. Operators deploying multiple tablets should leave the default (which correctly designates the first-paired device as audio) or adjust via the fleet view.

The tablet reads `audio_enabled` from its device record after each sync and applies it immediately. If `audio_enabled` is false, the audio playback engine is disabled for the entire session; the visual display continues normally.

True primary-secondary tablet linking with shared journey state is deferred. The architecture accommodates it (journey state is already in local Room and can be projected to other devices via Supabase Realtime later), but the initial release does not implement it.

### 5.3 Minimum and Recommended Hardware Configurations

**Minimum (suitable for pilot or small buses):** One tablet near the driver, one Bluetooth speaker. Approximately £280–£320 in hardware per bus.

**Recommended for production (buses where 51% sightline isn't met by a single screen):** One tablet near the driver, one or two additional tablets in the passenger cabin running the same software independently, one Bluetooth speaker. Approximately £430–£620 per bus.

Even at the recommended configuration, hardware cost per bus is dramatically below the £5,000 incumbent bundles.

### 5.4 Why the Multi-Surface Architecture

A single tablet near the driver cannot satisfy the regulations on most buses. The rules require visible information from at least 51% of seats, all priority seats, and all wheelchair spaces. On anything larger than a minibus, a single screen near the front isn't visible from the back half. Multiple tablets solve this by placing displays where passengers actually sit. The dashboard-driven authoring and JWT-based device identity make managing a fleet of these displays straightforward — every tablet pairs to the same operator account, sees the same routes, and reports back to the same fleet view.

---

## 6. Online vs Offline Modes — Summary

This section summarises which functionality requires connectivity, since offline-first operation is a core architectural principle.

### 6.1 Fully Offline (No Wi-Fi, No Cellular)

The tablet's primary operating mode during a journey. Everything the driver and passengers need works without any network connection.

**What works offline:**
- Selecting a synced route and starting a journey
- All GPS tracking and automatic stop detection
- All audio announcements (pre-rendered files stored on-device; no network required for playback)
- The full passenger visual display, including the tube-map progress view
- All driver controls (manual stop advance, diversions, hail-and-ride, end journey)
- Kiosk mode, boot-on-startup, admin PIN
- Journey state recovery after a crash or reboot

**What does NOT work offline:**
- Receiving new or updated routes from the dashboard
- Reporting last-seen status (which affects the operator's fleet view)
- Receiving suspension/reactivation status

A tablet could operate offline for days or weeks and continue to run journeys correctly. The only thing it misses is route updates and status changes from the dashboard. On reconnection, the device syncs everything it missed.

### 6.2 Connected (Wi-Fi or Cellular)

When the device has internet access, sync activates automatically in the background.

**What happens when connected:**
- Routes added or modified on the dashboard download to the tablet
- The device's last-seen timestamp updates, propagating to the operator's fleet view
- If the operator was suspended, the tablet receives that status and locks the UI
- FCM push notifications can trigger immediate sync rather than waiting for the next periodic check

The connection is used only for sync — no real-time functionality depends on it. A journey in progress is never affected by connectivity changes.

### 6.3 Transitioning Between Modes

- If the device loses connection mid-sync, the partial sync is discarded and retried on next connection
- If the device gains connection during a journey, sync occurs in the background without affecting the journey
- The sync status indicator on the route list screen shows when the last successful sync occurred

---

## 7. NaPTAN Data Approach

### 7.1 What NaPTAN Is

NaPTAN (National Public Transport Access Nodes) is the UK's authoritative public transport reference dataset, published by the Department for Transport under the UK Open Government Licence. It contains every railway station and bus stop in the UK — approximately 400,000 records — with stable identifiers, names, localities, and WGS84 coordinates.

### 7.2 Where NaPTAN Lives in This System

**On Supabase:** A `naptan_stations` table holds the full dataset. The dashboard's route builder queries this table directly for station/stop search via Postgres full-text search. Because all operators query the same shared reference data (no multi-tenancy concern), this table is publicly readable by authenticated dashboard users.

**On the tablet:** The tablet holds no NaPTAN data. Stop names, coordinates, and NaPTAN IDs travel with routes: when the dashboard creates a route stop, these fields are copied into the `route_stops` row (see §7.4). The tablet reads only from its synced `route_stops` table during operation; no NaPTAN bundle or local NaPTAN table is needed.

### 7.3 NaPTAN Updates

NaPTAN data updates infrequently (a few changes per year nationally). Updates are delivered through:
- **Dashboard:** the system administrator updates the `naptan_stations` table in Supabase via SQL import. The dashboard immediately reflects the new data.

NaPTAN updates affect the dashboard only. Tablets hold no NaPTAN data and require no update when NaPTAN changes — stop data is already captured in `route_stops` at route-creation time.

### 7.4 Stop Data Travelling With Routes

When the dashboard creates a route stop, the stop's NaPTAN ID, name, CRS code, and coordinates are *copied* into the `route_stops` row at creation time. The route_stops table is not a foreign key to naptan_stations — it's a snapshot. This means routes survive NaPTAN database changes (a station rename or removal does not break a saved route), and tablets don't need a fresh NaPTAN bundle to use a synced route.

---

## 8. Non-Functional Requirements

### 8.1 Performance

**NFR-P-01:** The app shall trigger a next-stop announcement within 2 seconds of entering the stop's proximity radius.

**NFR-P-02:** The passenger display shall update within 500ms of any state change.

**NFR-P-03:** GPS position updates shall occur at minimum every 3 seconds during an active journey.

**NFR-P-04:** The app shall launch from cold boot to the route selection screen within 30 seconds on the minimum supported device. (This requirement is achievable from first launch — the first-launch NaPTAN import has been removed; there is no longer a blocking one-time import at first boot.)

**NFR-P-05:** Route selection from the local database shall take less than 1 second, even with 100+ stored routes.

**NFR-P-06:** The dashboard's NaPTAN search shall return results within 500ms for typical queries against the full ~400K-record dataset.

**NFR-P-07:** The dashboard shall load the route list view in under 2 seconds for an operator with up to 200 routes.

### 8.2 Reliability

**NFR-R-01:** The app shall not crash during an active journey. Unhandled exceptions are caught, logged, and recovered from gracefully.

**NFR-R-02:** The foreground GPS service shall survive for the duration of any journey (up to 8 hours).

**NFR-R-03:** If an audio file cannot be played mid-journey (file missing or corrupt), the app continues operating the visual display and logs an `AUDIO_PLAYBACK_ERROR` event to journey_events. The driver is notified via the control panel. The journey is not terminated.

**NFR-R-04:** If GPS signal is lost for more than 60 seconds during a journey, the app displays a subtle indicator (visible only to the driver, not passengers) and switches to manual-advance expectations.

**NFR-R-05:** The app recovers journey state after any unexpected restart or device reboot.

**NFR-R-06:** Dashboard uptime shall meet Vercel's free-tier SLA. Dashboard outages do not affect tablets in active journeys.

### 8.3 Security

**NFR-S-01:** Operator data is isolated in Supabase via Row-Level Security policies bound to JWT claims. No operator can read, modify, or delete another operator's routes or devices.

**NFR-S-02:** Admin PIN, device JWT, refresh token, and device secret are stored in EncryptedSharedPreferences on the tablet, never in plaintext.

**NFR-S-03:** All communication between tablets, dashboard, and Supabase uses HTTPS exclusively.

**NFR-S-04:** Pairing codes are single-use, expire after 10 minutes, and are deleted from the database on use.

**NFR-S-05:** Passwords are hashed by Supabase Auth using industry-standard methods (bcrypt or equivalent). The system never stores or transmits passwords in plaintext.

**NFR-S-06:** Device authentication for session recovery relies on a 256-bit cryptographically random device secret generated at pairing time. The secret is stored in EncryptedSharedPreferences on the tablet and as a SHA-256 hash in `devices.device_secret_hash` on the server. The `recover-device` endpoint requires both the Android ID (as a non-secret row-lookup key) and the plaintext device secret (authenticated against the stored hash). An attacker who knows only the Android ID — which is a readable device identifier available to any app on the device — cannot obtain a valid device session. The Android ID is not a credential; the device secret is.

**NFR-S-07 (Stolen-Tablet Threat):** A tablet in kiosk mode (FR-AT-46) has no conventional lock screen during operation. A physically stolen tablet therefore holds a live operator-scoped JWT with read access to that operator's routes and device list. Write access is not available to tablets (routes are dashboard-only in the initial release).

**Mitigation:** The fleet manager deactivates the stolen device from the dashboard via FR-WD-17. On the deactivated device's next sync attempt or JWT refresh, the server rejects the session and the device returns to the first-run setup screen.

**Known limitation:** Between physical theft and the operator noticing and performing deactivation, the stolen tablet retains read access to that operator's routes. This is a known and accepted limitation for a product in this cost class. Operators with higher security requirements are advised to supplement with an Android Enterprise solution that supports remote lock or wipe.

### 8.4 Compatibility

**NFR-C-01:** Tablet app targets minimum Android API level 26 (Android 8.0).

**NFR-C-02:** Tablet app functions correctly on screen sizes from 10 inches to 15 inches.

**NFR-C-03:** Tablet app supports both landscape and portrait, with landscape as the default for passenger-facing displays.

**NFR-C-04:** Tablet app functions with or without a SIM card / cellular data connection.

**NFR-C-05:** Dashboard renders correctly on modern desktop browsers (last two major versions of Chrome, Firefox, Safari, Edge). Mobile browser support is not required but the dashboard should remain functional on tablets and phones.

### 8.5 Accessibility

**NFR-A-01:** The passenger display meets all visual requirements of the Public Service Vehicles (Accessible Information) Regulations 2023 as detailed in the Compliance Mapping Matrix.

**NFR-A-02:** Audio announcements are rendered using a server-side voice selected for clear articulation and at a pace that allows comprehension, including for passengers with cognitive impairments. The rendering voice is consistent across all announcements and all tablets.

**NFR-A-03:** Audio output is compatible with Audio Frequency Induction Loop (hearing loop) systems when the tablet is connected to one via the bus's audio system.

**NFR-A-04:** The dashboard meets WCAG 2.1 AA accessibility guidelines for keyboard navigation, screen reader compatibility, and colour contrast.

---

## 9. MoSCoW Prioritisation

The product is built as a single coherent release. The MoSCoW list below describes what's in scope for that release versus what's deliberately out of scope.

### Must Have (Initial Release)

**Web Dashboard:**
- Operator self-signup with Supabase Auth
- Manual approval workflow with email notification
- Route creation, editing, deletion, listing
- NaPTAN search across full UK dataset
- Return route generation
- Device pairing code generation
- Device fleet view with online/offline status
- Device naming and deactivation

**Android Tablet:**
- First-run pairing via 6-digit code
- Device JWT acquisition and recovery
- Route browsing and selection (read-only)
- GPS-based automatic stop detection with strict sequential progression
- Per-stop configurable proximity radius (set per stop on the dashboard, 200m default)
- All required audio announcements via pre-rendered audio files (next stop, route, termination, hail-and-ride, diversion)
- Alert chime before regulated announcements
- Stylised tube-map route progress view
- 22mm minimum text rendering with screen calibration
- High-contrast visual display
- Audio-visual consistency
- Manual stop navigation and announcement triggers
- Audio output handling with Bluetooth fallback
- Bluetooth keep-alive pings
- Driver control panel with auto-dismiss
- Emergency mute toggle
- Journey auto-timeout
- Kiosk mode (Level 1 soft kiosk — screen pinning and default launcher registration)
- Boot-on-startup
- Screen wakelock during journeys
- Admin PIN unlock and admin menu
- Foreground GPS service with battery optimisation bypass
- Offline-first operation with sync
- FCM token registration and push notification receipt for instant route sync
- Account suspension lock
- Journey state recovery
- Journey event logging (local only)

**Backend:**
- Supabase project with operator-scoped RLS
- Edge Functions for `pair-device`, `recover-device`, and `generate-pairing-code`
- Edge Function `render-route-audio` for server-side audio rendering at route-save time
- Supabase Storage bucket `route-audio` for per-route pre-rendered audio files
- Server-side timestamp triggers
- Atomic route + stops upsert RPC
- Cursor-based sync RPC
- NaPTAN data table with full-text search
- Edge Function to send FCM push notifications to operator devices on route change

### Should Have (Initial Release if Time Permits)

- Sync status indicator with last-sync timestamp on tablet route list
- Driver control panel repeat-last-announcement button
- Announcement overlay fade animations
- Dashboard search/filter on the route list view

### Could Have (Future Releases)

- Stripe billing integration
- Team / multi-user accounts on the dashboard
- Journey analytics and reporting
- Custom stop creation (drop-pin at GPS location)
- Operator branding (logo on idle screen)
- Per-stop diversion audio (naming specific affected stops in speech, for visually impaired passengers — deferred; generic diversion audio is in initial release)
- Primary/secondary tablet linking with shared journey state
- Kiosk Level 2 (Device Owner mode / Android Lock Task Mode) — full tamper-proof kiosk for production deployments; architecture accommodates it without redesign, only provisioning differs
- Mobile-side route creation for offline route edits
- Automated NaPTAN updates over-the-air
- Welsh language support
- Google Play Store distribution

### Won't Have (Out of Scope)

- Turn-by-turn driving navigation for the driver
- Real-time passenger journey planning
- Integration with ticketing or payment systems
- Live vehicle tracking for passengers
- Multi-language announcements (English only)
- Integration with existing transit data feeds (GTFS, SIRI)
- Mapping-API-based journey visualisation
- Full UK addressing data beyond NaPTAN

---

## 10. Legal Requirements Summary

The Public Service Vehicles (Accessible Information) Regulations 2023 require the following information in both audible and visible formats on local bus services:

1. **Route and destination** — audible and visible at journey start and after each stop
2. **Next stop** — audible and visible as the bus approaches each stop
3. **Termination** — audible and visible announcement at the final stop, preceded by an alert
4. **Diversions** — audible and visible announcement when a diversion begins, preceded by an alert
5. **Hail and ride** — audible and visible announcement at start and end of hail-and-ride sections, preceded by an alert

**Visual display requirements:**
- Characters at least 22mm in height
- No text exclusively in capital letters
- Visible from at least 51% of seats on each deck
- Visible from all priority seats
- Visible from all wheelchair spaces

**Audio requirements:**
- Audible to passengers in any seat or wheelchair space
- Compatible with hearing aids via Audio Frequency Induction Loop when seated in a priority seat or wheelchair space
- Audio and visual information must be consistent

Physical screen placement, sightline compliance, and hearing loop installation are the bus operator's responsibility. The software provides the compliant audio and visual content; the operator ensures the hardware delivers it correctly.

The detailed clause-by-clause mapping is provided in the Compliance Mapping Matrix document.

---

## 11. Assumptions and Dependencies

### 11.1 Assumptions

1. The tablet has a functioning GPS receiver.
2. The tablet is connected to a Bluetooth speaker or wired audio output of sufficient volume for the bus. The speaker is mounted out of passenger reach.
3. The bus operator is responsible for physical screen placement meeting sightline requirements.
4. NaPTAN data for UK railway stations and bus stops is accurate and up to date.
5. Routes will typically have 3–15 stops.
6. Dashboard users have basic web literacy.
7. Drivers have basic familiarity with touchscreen devices.
8. The tablet is connected to vehicle power during operation, not running on battery.
9. Tablets are Google Mobile Services (GMS) certified to ensure FusedLocationProviderClient and FCM function correctly.
10. Tablets ideally feature a "battery protect" mode (cap charge at 80%) or use cycling chargers, to prevent battery swelling from continuous 100% charging combined with screen wakelocks.

### 11.2 Dependencies

1. **Google Play Services** — required for FusedLocationProviderClient and FCM.
2. **Supabase** — backend for auth, database, Edge Functions, Storage, and FCM relay. Subject to Supabase's SLA.
3. **Server-side TTS API** (e.g., Google Cloud Text-to-Speech) — used by the `render-route-audio` Edge Function to render announcement audio server-side. This is a dashboard/server-side dependency only; the tablet has no TTS dependency.
4. **Vercel** — dashboard hosting. Subject to Vercel's free-tier limits.
5. **Next.js** — dashboard framework.
6. **NaPTAN open data** — published by the Department for Transport under UK Open Government Licence.

---

## 12. Success Metrics

### Pilot Success (first 4 weeks on a real bus)

- Zero crashes during active journeys
- Announcements trigger at the correct stop with at least 95% accuracy
- Driver can start a journey from a powered-on tablet in under 30 seconds
- Fleet manager can create a new route on the dashboard and have it appear on a tablet within 5 minutes (requires FCM — a Must-Have feature, §9 — for tablets that are already online; ConnectivityManager handles the offline→online case)
- Positive qualitative feedback from driver, fleet manager, and at least one passenger

### Product-Market Fit (first 3 paying customers)

- At least 3 operators subscribed and actively using the product
- Monthly churn below 10%
- Total cost of ownership demonstrably lower than incumbent solutions

---

## 13. Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| GPS signal loss in urban canyons or tunnels | Missed stop announcements | High | Manual advance fallback, GPS loss indicator to driver, journey state recovery |
| Audio rendering voice quality | Announcements sound robotic or unclear | Low | Server-side rendering voice is selected and tested at development time; quality is consistent and controllable regardless of tablet hardware |
| Android OEM battery optimisation kills foreground service | GPS tracking stops mid-journey | Medium | REQUEST_IGNORE_BATTERY_OPTIMIZATIONS, foreground service with notification, approved hardware list |
| Operator installs app on unsupported device | App malfunctions, blamed on software | Medium | Documented minimum device specification; strict GMS requirement |
| Pairing code phishing or social engineering | Unauthorised tablet pairs to operator account | Low | Codes are single-use, expire in 10 minutes, and require dashboard login to generate |
| Regulatory interpretation changes | Features become non-compliant | Low | Compliance Mapping Matrix enables rapid gap analysis; modular architecture allows targeted updates |
| Supabase service disruption | Cannot sync new routes; pairings fail | Low | Offline-first design means active routes are unaffected; outages affect onboarding only |
| Dashboard service disruption | Cannot manage routes or pair new devices | Low | Tablets continue operating from cached data |
| Operator subscribes via self-signup but has no intention of paying | Wasted resources, system abuse | Medium | Manual approval gate before any tablet functions |

---

## 14. Out of Scope for This Document

- Detailed database schema (see Data Architecture Document)
- Sync algorithm internals (see Data Architecture Document)
- Edge Function implementations (see Data Architecture Document)
- Clause-by-clause regulatory mapping (see Compliance Mapping Matrix)
- Pricing strategy and contract terms
- Marketing and go-to-market strategy
- Detailed Android implementation conventions (see CLAUDE.md)
- Workflow conventions for the architect/builder split (see WORKFLOW.md)
