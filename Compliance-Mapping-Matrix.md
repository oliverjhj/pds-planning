# Compliance Mapping Matrix
# Passenger Display System (PDS) — Accessible Information Regulations

**Version:** 3.8
**Last Updated:** May 2026
**Legislation:** The Public Service Vehicles (Accessible Information) Regulations 2023 (SI 2023/715)
**Source:** https://www.legislation.gov.uk/uksi/2023/715/made
**PRD Alignment:** PRD v3.8

## Changelog

### v3.8 (May 2026)
Round-2 post-adversarial-review re-planning pass, item 1 of 4 (audio pipeline overhaul). Voice locked to Google Cloud TTS `en-GB-Neural2-B`. Reg 13(4) frequency-range row tightened to a BLOCKING WORK pre-deployment verification task rather than an assertion of existing compliance.
- Reg 13(4): Voice locked. Frequency verification stated as a one-off pre-deployment task (spectrum-analyser check on a rendered sample). Row now explicit about the work that must complete before production deployment.
- Reg 12(1)(b): Consistency argument strengthened — locked voice means consistency is structural, not configurable.
- Responsibility Boundary Summary: Audio frequency row updated to reference locked voice.
- PRD References updated to include FR-WD-20.

### v3.7 (May 2026)
Post-adversarial-review re-planning pass, item 7 of 7 (compliance, WORKFLOW, smaller items, and campaign close-out).
- Reg 7(1)(a) / Reg 9(1): Added note that FR-AT-40 non-occlusion constraint ensures next-stop name and route name remain visible even when driver panel is open.
- Regs 8(2), 10(2)(b), 11(2)(b), 11(5)(b): Added reference to new visual alert FR-AT-65 (500ms high-contrast screen flash) as the visual component of the combined alert. PRD References updated.
- Reg 12(1)(b): Updated consistency argument to cover multi-tablet audio designation — single primary audio tablet per bus; `audio_enabled` flag prevents overlapping audio.
- Reg 13(4): Tightened to clearly distinguish guarantee (rendered MP3 frequency content, verified at development time) from expectation (operator speaker fidelity).
- Alert Tone Requirements section: Added "Visual Alert" column to table; updated prose to describe FR-AT-65 as co-equal element with FR-AT-27.

### v3.3 (May 2026)
Post-adversarial-review re-planning pass, item 3 of 7 (audio architecture — pre-rendered files).
- Reg 8(2): Updated to reference pre-rendered `termination.mp3` (bundled APK asset); removed TTS reference.
- Reg 10(1): Rewritten. Compliance argument now rests on audio+visual pair: generic diversion audio signals the state; tube-map strikethrough names the specific affected stops. Per-stop audio naming deferred to future release.
- Reg 10(2)(b): Updated to reference pre-rendered `diversion_start.mp3`.
- Reg 11(2)(b): Updated to reference pre-rendered `hail_and_ride_start.mp3`.
- Reg 11(5)(b): Updated to reference pre-rendered `hail_and_ride_end.mp3`.
- Reg 12(1)(a): Updated "TTS audio" → "pre-rendered audio".
- Reg 12(1)(b): Major rewrite. Consistency argument now rests on server-rendered audio files — identical file bytes distributed to all tablets, with no per-device variation. Substantially stronger than the previous on-device TTS argument.
- Reg 13(4): Major rewrite. Frequency-range claim now rests on a controlled, testable server-side rendering environment rather than on assumptions about on-device TTS behaviour.
- Alert Tone Requirements section: Updated TTS references to pre-rendered audio throughout.
- Responsibility Boundary Summary: Updated audio frequency and audio/visual consistency rows.
- Compliance Note 1: Updated alert-before-TTS → alert-before-audio-file.

### v3.2 (May 2026)
Post-adversarial-review re-planning pass, item 2 of 7 (hail-and-ride and diversion data model).
- Reg 10(1): Rewritten. Compliance argument now rests on the diversion skip-range mechanism and dynamic stop-naming announcement, directly addressing the "where" requirement. GPS state machine auto-skips marked stops.
- Reg 10(2)(a)(i): Updated PRD reference to include FR-AT-42.
- Reg 11 section intro: Added note that hail-and-ride sections are a first-class data concept via `route_stops.segment_type`.
- Reg 11(1), 11(2)(a), 11(3), 11(4), 11(5)(a): Rewritten. Compliance now rests on automatic GPS segment-boundary triggers from route data, not driver judgment. Manual buttons retained as fallback.
- Responsibility Boundary Summary: Updated diversion/H&R row; added two new rows.

### v3.1 (May 2026)
Post-adversarial-review re-planning pass, item 1 of 7 (stop detection).
- Reg 7(2)(b): Updated PDS Feature to reflect per-stop proximity radius and GPS accuracy gate.
- Reg 9(2)(a): Updated PDS Feature to describe two-stop look-ahead as a missed-detection tolerance mechanism within strict sequential progression.
- Reg 9(2)(b): Full rewrite. Timing argument now rests on per-stop `proximity_radius_meters` (set on dashboard, not admin settings). Added GPS accuracy gate note. PRD reference updated from FR-AT-49 to FR-WD-08.

---

## Purpose

This document maps every applicable clause of the Public Service Vehicles (Accessible Information) Regulations 2023 to a specific feature, behaviour, or responsibility boundary in the Passenger Display System. It serves two purposes: an internal development checklist to ensure nothing is missed during implementation, and an evidence document that can be presented to bus operators to demonstrate how the software meets each legal requirement.

**Important distinction:** Some requirements are satisfied by the software, some by the operator's hardware setup (screen placement, speaker volume, hearing loop), some by the operator's content (accurate route names, stop names), and some are shared. This matrix clearly labels who is responsible for each clause.

---

## Regulation 7 — Route Information

Passengers must be given route identification and destination/direction information at each stopping place.

| Reg. Clause | Requirement Summary | PDS Feature | Responsibility | PRD Reference |
|---|---|---|---|---|
| 7(1)(a) | Passengers must be given the name, number, or other label used to designate the route | The passenger display shows the route name/number at all times during a journey. Audio announcement at each stop: "This bus is the [Route Name] service to [Final Stop]." The driver control panel (FR-AT-40) opens as an overlay over the lower portion of the screen only; the passenger information area (top two-thirds: route name, next-stop name) is never occluded by the driver panel, ensuring route information remains continuously visible even when the driver is interacting with the panel. | Software | FR-AT-12, FR-AT-22, FR-AT-40, FR-WD-10 |
| 7(1)(b)(i) | Passengers must be given the name of the final scheduled stopping place | The final stop name is displayed on the passenger screen and included in the route announcement audio at each stop. | Software | FR-AT-12, FR-AT-22 |
| 7(1)(b)(ii) | Alternatively, the direction of travel may be given | The route metadata includes a direction label (e.g., "Outbound" / "Return") displayed on the passenger screen. The system provides the final stop name (7(1)(b)(i)) as the primary method, satisfying this requirement. | Software | FR-WD-10, FR-AT-12 |
| 7(2)(a) | Information must be provided at each stopping place on the route | The route and destination announcement is triggered automatically at each stop when the bus arrives (GPS proximity trigger or manual advance). | Software | FR-AT-13, FR-AT-22 |
| 7(2)(b) | Information must begin when the doors are open at that stopping place | The announcement triggers as the bus enters the stop's proximity radius (set per stop on the dashboard, defaulting to 200m), which is before or as the bus arrives and doors open. GPS fixes whose accuracy estimate exceeds the stop's proximity radius are discarded, preventing a bad fix from triggering an erroneous or premature announcement. The driver can also trigger manually. | Software | FR-AT-13, FR-AT-15 |
| 7(3) | Provision of information need not be completed before the service departs the stopping place | The system begins the announcement on approach/arrival. Audio playback continues even if the bus departs before the announcement finishes. | Software | FR-AT-13 |

---

## Regulation 8 — Route Termination Information

Passengers must be informed when the service reaches its final stopping place, preceded by an alert.

| Reg. Clause | Requirement Summary | PDS Feature | Responsibility | PRD Reference |
|---|---|---|---|---|
| 8(1) | Passengers must be informed when the service reaches the final stopping place on its route | At the final stop, the app plays a termination announcement ("This service terminates here. All change, please.") and displays a prominent termination message on the passenger screen. | Software | FR-AT-16, FR-AT-24 |
| 8(2) | The termination information must be immediately preceded by an alert | A short audible chime (FR-AT-27) plays immediately before the pre-rendered termination announcement (`termination.mp3`, a bundled APK asset). Simultaneously, a 500ms high-contrast screen flash (FR-AT-65) fires as the visual alert component, before the announcement overlay text (FR-AT-38) appears. Together, FR-AT-27 and FR-AT-65 constitute the combined audible-and-visual alert required by this clause. See Alert Tone Requirements section below. | Software | FR-AT-24, FR-AT-27, FR-AT-65 |

---

## Regulation 9 — Stopping Place Information

Passengers must be informed of the next scheduled stopping place.

| Reg. Clause | Requirement Summary | PDS Feature | Responsibility | PRD Reference |
|---|---|---|---|---|
| 9(1) | Passengers must be informed of the next scheduled stopping place on the route | The app announces "Next stop: [Stop Name]" via audio and displays the next stop prominently on the passenger screen. The driver control panel (FR-AT-40) non-occlusion constraint ensures the next-stop name in the passenger information area remains visible at all times, including when the driver panel is open (e.g., to trigger a hail-and-ride or diversion announcement near a stop). | Software | FR-AT-12, FR-AT-23, FR-AT-40 |
| 9(2)(a) | Information must be provided after the preceding stopping place | The next stop announcement triggers after the bus departs the previous stop (exits that stop's proximity radius) and enters the next stop's proximity radius. Strict sequential progression with two-stop look-ahead tolerance ensures correct ordering: the app monitors the next expected stop (N) and the one after it (N+1). If the bus enters N normally, the announcement fires for N. If the bus enters N+1 without N having been registered, N is logged as passed without detection and the announcement fires for N+1. The stop sequence from route creation is the sole authority — the app never selects stops by geographic proximity. | Software | FR-AT-13, FR-AT-14 |
| 9(2)(b) | Information must be provided in sufficient time before reaching the next stop to enable passengers to leave the vehicle | Each stop has its own proximity radius stored in `route_stops.proximity_radius_meters` (set by the operator in the dashboard route builder, defaulting to 200m). The announcement triggers when the bus enters this radius, giving passengers time to prepare before the bus arrives. Operators can configure a larger radius for motorway-services stops and a smaller one for dense town-centre stops. GPS fixes whose accuracy estimate is worse than the stop's proximity radius are rejected by the accuracy gate and cannot trigger the announcement — this prevents a bad fix from firing an announcement at the wrong location or too early. | Software | FR-AT-13, FR-WD-08 |

---

## Regulation 10 — Diversion Information

Passengers must be informed when the service is diverted from its scheduled route.

| Reg. Clause | Requirement Summary | PDS Feature | Responsibility | PRD Reference |
|---|---|---|---|---|
| 10(1) | Passengers must be informed where the service is being diverted, resulting in inability to stop at one or more scheduled stopping places | The driver uses the diversion stop selector (FR-AT-42) to mark upcoming affected stops as skipped, then triggers the diversion announcement (FR-AT-41). The spoken audio announcement is a fixed pre-rendered phrase: "This service is on diversion. Please check the display for affected stops." (`diversion_start.mp3`, a bundled APK asset). Simultaneously, the passenger display's tube-map view shows each specific skipped stop by name with a strikethrough marker. The regulation's requirement to inform passengers *where* the diversion affects service is satisfied by the audio+visual combination: the audio signals that a diversion is active; the visual display names each affected stop. This is an honest split — the audio is generic, the specificity is visual. The GPS state machine automatically skips over stops in `journey_skipped_stops`, removing the need for the driver to manually advance past every unvisited stop. **Note:** Per-stop audio naming (speaking the specific affected stop names) is a candidate future enhancement for visually impaired passengers and has been deliberately deferred from the initial release. | Software + Driver action | FR-AT-25, FR-AT-41, FR-AT-42 |
| 10(2)(a)(i) | Where the driver knows about the diversion before the last stop prior to it, information must be provided in sufficient time for passengers to leave at that stop | The driver can open the diversion stop selector and trigger the announcement at any time, including before reaching the last stop prior to the diversion. The system provides the mechanism; the driver's judgment determines timing. | Software + Driver action | FR-AT-41, FR-AT-42 |
| 10(2)(a)(ii) | Where the driver does not know in advance, information must be provided at or as soon as possible after commencement of the diversion | The diversion button is available at all times during an active journey for immediate use. | Software + Driver action | FR-AT-41 |
| 10(2)(b) | The diversion information must be immediately preceded by an alert | A short audible chime (FR-AT-27) plays immediately before the pre-rendered diversion announcement (`diversion_start.mp3`, a bundled APK asset). Simultaneously, a 500ms high-contrast screen flash (FR-AT-65) fires as the visual alert component. See Alert Tone Requirements section below. | Software | FR-AT-25, FR-AT-27, FR-AT-65 |

**Responsibility note:** The regulations require the information to be provided. The PDS provides the mechanism (button and announcement). The driver is responsible for triggering it at the appropriate time. The operator is responsible for training drivers to use this feature.

---

## Regulation 11 — Hail and Ride Information

Passengers must be informed about hail and ride sections of the route. PDS routes represent hail-and-ride sections as a first-class data concept via `route_stops.segment_type = 'hail_and_ride'`. A contiguous run of such stops constitutes a hail-and-ride section. The GPS state machine automatically fires the required announcements at section boundaries, with manual driver buttons retained as fallback.

| Reg. Clause | Requirement Summary | PDS Feature | Responsibility | PRD Reference |
|---|---|---|---|---|
| 11(1) | Where a hail and ride service is preceded by scheduled stops, passengers must be informed it will be commencing | The GPS state machine detects the approaching hail-and-ride section boundary from `route_stops.segment_type`. When it announces the last scheduled stop before the section, it automatically fires: "You are now entering a hail and ride section. Please signal the driver if you wish to alight." A manual fallback button is available in case GPS fails. The section boundary is defined by the route data, not by driver judgment. | Software (automatic GPS trigger); Driver (manual fallback) | FR-AT-13, FR-AT-26, FR-AT-41 |
| 11(2)(a) | This information must be provided at the last scheduled stopping place prior to the hail and ride service | The automatic trigger fires when the GPS state machine announces the last scheduled stop prior to the hail-and-ride section — i.e., at the correct regulatory timing point. This is determined from the segment data in `route_stops`, not by driver judgment. | Software (automatic GPS trigger); Driver (manual fallback if GPS fails) | FR-AT-13, FR-AT-26 |
| 11(2)(b) | The information must be immediately preceded by an alert | A short audible chime (FR-AT-27) plays immediately before the pre-rendered hail-and-ride start announcement (`hail_and_ride_start.mp3`, a bundled APK asset), whether triggered automatically or manually. Simultaneously, a 500ms high-contrast screen flash (FR-AT-65) fires as the visual alert component. See Alert Tone Requirements section below. | Software | FR-AT-26, FR-AT-27, FR-AT-65 |
| 11(3) | Where the entire service is hail and ride, passengers must be informed at the start of the route | At journey start (FR-AT-11), if all stops in the route have `segment_type = 'hail_and_ride'`, the system automatically fires the hail-and-ride start announcement before GPS monitoring begins. The driver does not need to manually trigger it. | Software (automatic at journey start); Driver (manual fallback) | FR-AT-11, FR-AT-13, FR-AT-26 |
| 11(4) | Where hail and ride is followed by scheduled stops, passengers must be informed it is ending | The GPS state machine detects the approaching end of the hail-and-ride section from `route_stops.segment_type`. When the first scheduled stop after the section enters its proximity radius, the system automatically fires: "You are now leaving the hail and ride section." A manual fallback button is available. | Software (automatic GPS trigger); Driver (manual fallback) | FR-AT-13, FR-AT-26, FR-AT-41 |
| 11(5)(a) | The end information must be provided before the end of the hail and ride service, in sufficient time before the first scheduled stop after it | The automatic trigger fires before the "Next stop" announcement for the first scheduled stop after the hail-and-ride section — i.e., in sufficient time before that stop. The route data (`segment_type`) determines the boundary precisely; the driver does not need to estimate timing. | Software (automatic GPS trigger); Driver (manual fallback if GPS fails) | FR-AT-13, FR-AT-26 |
| 11(5)(b) | The end information must be immediately preceded by an alert | A short audible chime (FR-AT-27) plays immediately before the pre-rendered hail-and-ride end announcement (`hail_and_ride_end.mp3`, a bundled APK asset), whether triggered automatically or manually. Simultaneously, a 500ms high-contrast screen flash (FR-AT-65) fires as the visual alert component. See Alert Tone Requirements section below. | Software | FR-AT-26, FR-AT-27, FR-AT-65 |

---

## Regulation 12 — General Requirements (Audio and Visual)

Information must be provided in both audio and visual form, and must be consistent.

| Reg. Clause | Requirement Summary | PDS Feature | Responsibility | PRD Reference |
|---|---|---|---|---|
| 12(1)(a) | Relevant information must be provided in audio and visual form | Every announcement (route, next stop, termination, diversion, hail-and-ride) is delivered simultaneously as both pre-rendered audio and on-screen visual display. | Software | FR-AT-32 |
| 12(1)(b) | The audio and visual forms must be consistent with one another | Audio and visual content are generated from the same data source (the route and stop database), and audio is rendered server-side using a **single locked voice** (`en-GB-Neural2-B`, Google Cloud TTS — see Reg 13(4) and PRD FR-WD-20). All tablets in the operator's fleet — regardless of model, Android version, or manufacturer — play identical audio file bytes for any given announcement. Consistency is structural: there is no per-device variation in voice quality, accent, pronunciation, or audio rendering. The locked voice means consistency is invariant under any future per-operator or per-deployment configuration — there is no such configuration, by design. The only way to change the voice is a deliberate compliance event (re-running the Reg 13(4) verification and updating the code-level voice constant). Audio files are rendered server-side by the `audio-render-worker` job (Data Architecture §4.6) at route-save time and distributed to every tablet via sync. **Multi-tablet audio designation:** in deployments with multiple tablets on one bus, exactly one tablet (the `audio_enabled = true` device, configured from the dashboard fleet view) produces audio output; all others run in display-only mode. This prevents overlapping announcements from multiple tablets playing the same file slightly out of sync, which would violate the consistency requirement. The visual display on all tablets remains consistent regardless of audio designation. The consistency guarantee covers: identical audio bytes, single locked voice across all announcements and all operators, single audio source per bus, and audio-visual content parity across the fleet. | Software | FR-AT-28, FR-AT-32, FR-WD-20 |
| 12(2) | The operator must not require passengers to use a personal electronic device to receive information | All information is displayed on the bus's tablet screens and played through the bus's speaker. No smartphone app, QR code, or personal device is needed. The system is fully self-contained. | Software + Operator hardware | FR-AT-12, FR-AT-29 |

---

## Regulation 13 — Audio Requirements

Requirements for volume, frequency range, and hearing aid compatibility.

| Reg. Clause | Requirement Summary | PDS Feature | Responsibility | PRD Reference |
|---|---|---|---|---|
| 13(2) | Without adaptive volume: audio must be at least 3dB louder than pre-measured ambient volume, and no louder than 84dB | The app provides volume control with a minimum floor to prevent muting, physical volume button interception, and an emergency mute toggle. The operator is responsible for measuring ambient volume and setting the speaker volume appropriately during installation. The app cannot measure or enforce absolute dB levels — this depends on the speaker hardware, placement, and bus acoustics. | Shared: Software provides volume control; Operator sets and verifies absolute levels | FR-AT-44 |
| 13(3) | With adaptive volume: audio must be at least 3dB louder than live-measured ambient volume, no louder than 84dB | Adaptive volume control (automatically varying volume based on live ambient levels) is not in scope for this release. This would require a microphone-based ambient noise measurement system. If an operator requires adaptive volume, this is a future-release feature. | Operator (current); Software (future) | N/A (future feature) |
| 13(4) | Audio must be within the frequency range of 300Hz to 3000Hz | Audio is rendered server-side using **Google Cloud Text-to-Speech, voice `en-GB-Neural2-B`** (see PRD FR-WD-20 and Data Architecture §4.6). The voice is locked — it is not configurable per operator, per route, or per deployment. The same voice is used both for route-specific files rendered by the `audio-render-worker` and for the bundled fixed-announcement files (termination, hail-and-ride start/end, diversion start/end) shipped in the APK. **BLOCKING WORK — voice locked to `en-GB-Neural2-B`; frequency verification required pre-deployment.** Before serving production traffic, the system administrator must: (1) render a representative sample sentence (e.g. "Next stop: Newcastle Central.") through the production pipeline; (2) pass the resulting MP3 through a spectrum analyser (Audacity's Spectrum view or any FFT-based audio tool); (3) confirm the energy of the speech content sits within 300Hz–3000Hz — sibilants and fricatives above 3000Hz are inherent to human speech and are acceptable provided the core speech content (the regulation's testable element) is in band; (4) record the result (waveform screenshot + written confirmation) in the operations runbook and sign off. This is a **one-off pre-deployment task**, not a per-route or per-deployment check, because the voice is locked: any future change to the voice is a deliberate compliance event that re-triggers this verification. If the verification fails the chosen voice, a different Google TTS voice must be selected, the voice lock updated in code, and the verification re-run — this is a Case 1 re-planning trigger. **What the software guarantees once verified:** every MP3 served to tablets contains speech audio produced by the verified voice in the verified rendering configuration; bytes are identical across the entire fleet (no per-device variation). **What the software does not guarantee:** that operator speaker hardware reproduces the 300Hz–3000Hz band without distortion or filtering — that remains the operator's responsibility. | Shared: Software produces verified-frequency audio (verification is a blocking pre-deployment task); Operator ensures speaker hardware fidelity | FR-AT-28, FR-WD-20 |
| 13(5) | Audio must be capable of being heard by a hearing impaired person using a hearing aid in a priority seat or wheelchair space | This requires an Audio Frequency Induction Loop (hearing loop) installed in the bus, connected to the same audio output as the announcement speaker. The app's audio output is routed through the tablet's audio jack or Bluetooth, which the operator connects to the bus's audio system including the hearing loop. The software generates the audio; the operator provides the hearing loop hardware. | Operator hardware | NFR-A-03 |

**Operator guidance note:** For Regulation 13 compliance, the bus operator must: (1) measure ambient volume on their vehicles, (2) set the speaker volume to at least 3dB above ambient but below 84dB, (3) ensure the speaker reproduces frequencies in the 300Hz–3000Hz range without distortion, and (4) install an audio induction loop connected to the announcement audio output for hearing aid users in priority seats and wheelchair spaces.

---

## Regulation 14 — Visual Requirements

Requirements for display visibility, text size, case, and contrast.

| Reg. Clause | Requirement Summary | PDS Feature | Responsibility | PRD Reference |
|---|---|---|---|---|
| 14(2)(a) | Uninterrupted line of sight between at least one display and at least 51% of passenger seats (per deck) | The software displays information on the tablet screen(s). The operator is responsible for physically mounting screens in positions that achieve 51% sightline coverage. Multiple independent tablets running the same software may be deployed for larger buses. | Operator hardware placement | Section 5 (Physical Device Placement) |
| 14(2)(b) | Uninterrupted line of sight from each priority seat | The operator must position at least one tablet screen visible from every priority seat. | Operator hardware placement | Section 5.2 |
| 14(3)(a) | For vehicles first used before 1 Oct 2024: line of sight from each forward-facing wheelchair space | The operator must position screens visible from all forward-facing wheelchair spaces. | Operator hardware placement | Section 5.2 |
| 14(3)(b) | For vehicles first used on or after 1 Oct 2024: line of sight from each wheelchair space (including rearward-facing) | For rearward-facing wheelchair spaces, a forward-facing screen at the rear of the bus is needed. Operators can achieve this by deploying an additional tablet running the same software, mounted facing forward at the rear of the bus. | Operator hardware placement + Software | Section 5.2 |
| 14(4) | Character height of text must be at least 22 millimetres | The app enforces 22mm minimum text height for all passenger-facing information. A screen calibration feature (bank card calibration in admin settings) ensures accurate physical sizing even on tablets with inaccurate firmware DPI values. | Software | FR-AT-34, FR-AT-35 |
| 14(5)(a) | No word may be displayed in capital letters only | The app enforces mixed case throughout all passenger-facing text. No text rendering path produces all-caps output. | Software | FR-AT-36 |
| 14(5)(b) | There must be contrast between text and background | The passenger display uses high-contrast colour schemes (light text on dark background or vice versa) designed for readability in varying lighting conditions. | Software | FR-AT-37 |

---

## Alert Tone Requirements (Cross-Regulation Implementation Requirement)

Regulations 8(2), 10(2)(b), 11(2)(b), and 11(5)(b) all require that certain announcements be "immediately preceded by an alert." This requirement spans multiple clauses but is implemented as a single coherent feature.

| Announcement Type | Regulation | Alert Required | Audio Alert (FR-AT-27) | Visual Alert (FR-AT-65) |
|---|---|---|---|---|
| Route termination | 8(2) | Yes — legally required | Short audible chime immediately before termination announcement | 500ms high-contrast screen flash, simultaneous with chime |
| Diversion start | 10(2)(b) | Yes — legally required | Short audible chime immediately before diversion announcement | 500ms high-contrast screen flash, simultaneous with chime |
| Hail and ride start | 11(2)(b) | Yes — legally required | Short audible chime immediately before H&R start announcement | 500ms high-contrast screen flash, simultaneous with chime |
| Hail and ride end | 11(5)(b) | Yes — legally required | Short audible chime immediately before H&R end announcement | 500ms high-contrast screen flash, simultaneous with chime |
| Next stop | 9 (no alert clause) | No — not required by law | None. Announcement plays directly. | None. |
| Route and destination | 7 (no alert clause) | No — not required by law | None. Announcement plays directly. | None. |

**Implementation specification:** The alert has two co-equal components — audio and visual — that fire simultaneously:

**Audio component (FR-AT-27):** A consistent, recognisable bundled audio file — a short two-tone or similar pre-recorded sound, distinct from the announcement audio, under 1 second long. The same chime is used for all four alert-required announcements. It plays at the same volume as the subsequent announcement. All announcement audio following the chime is pre-rendered (FR-AT-28).

**Visual component (FR-AT-65):** A 500ms high-contrast screen flash — an inverted or strongly contrasting colour relative to the resting display — that fires at the same instant as the chime begins. The announcement overlay text (FR-AT-38) appears immediately after the flash completes, while the chime audio is still playing. The flash is distinct from both the normal display and the overlay, making it a clearly perceptible visual event even for passengers who did not hear the chime.

Together, FR-AT-27 and FR-AT-65 constitute the combined alert required by Regulations 8(2), 10(2)(b), 11(2)(b), and 11(5)(b). This rule must be enforced by the implementation and is codified in CLAUDE.md as an inviolable architectural rule.

**Audio concurrency note:** If a manual announcement (e.g., diversion) is triggered while an automatic announcement is playing, the manual announcement interrupts and replaces the current audio (FR-AT-33). The alert chime for the new announcement plays first, followed by the pre-rendered announcement audio file.

---

## Responsibility Boundary Summary

This table provides a clear summary of what the software handles versus what the bus operator is responsible for. This is critical for operator contracts and liability.

| Requirement Area | Software (PDS) Responsibility | Operator Responsibility |
|---|---|---|
| Route, destination, next stop information content | Generates and displays/announces all required information automatically via GPS triggers | Ensures drivers are trained to use the system |
| Accuracy of route content (route name, stop names, stop ordering) | Faithfully displays and announces the data that was entered into the dashboard | Inputs accurate route data via the web dashboard; reviews routes before deploying them to drivers |
| Termination announcement | Provides announcement mechanism with correct content and alert tone | Ensures drivers trigger termination announcement at journey end |
| Hail-and-ride section start and end announcements | Automatically triggers at segment boundaries defined in route data; manual fallback buttons available for GPS failure | Ensures route data accurately marks hail-and-ride stops in the dashboard route builder |
| Diversion announcements with affected stop names | Provides stop selector and dynamic announcement content naming affected stops; GPS state machine automatically skips past stops marked in `journey_skipped_stops` | Driver marks which upcoming stops are skipped and triggers the diversion announcement at the appropriate time |
| Audio and visual consistency | Guarantees consistency by (a) generating both audio and visual from the same data source, and (b) distributing identical pre-rendered audio files to all tablets — no per-device synthesis variation | None (handled entirely by software) |
| No personal device requirement | All information displayed on bus-mounted screens and speakers | Provides and mounts the tablet(s) and speaker |
| Audio volume (3dB above ambient, max 84dB) | Provides software volume control with minimum floor and physical button interception | Measures ambient volume, sets speaker level, verifies compliance, mounts speaker out of passenger reach |
| Audio frequency (300Hz–3000Hz) | Renders audio server-side using Google Cloud TTS voice `en-GB-Neural2-B` (locked). Frequency content of the rendered output must be verified pre-deployment with a spectrum-analyser pass on a sample sentence; re-verification is required only if the locked voice ever changes (which is itself a deliberate compliance event). | Ensures speaker hardware does not distort or filter frequencies |
| Hearing loop compatibility | Outputs audio via standard audio connection | Installs audio induction loop hardware, connects to audio output |
| 51% sightline coverage | Displays information on screen(s); supports multi-tablet deployment | Mounts screens in compliant positions |
| Priority seat and wheelchair space visibility | Provides software that runs independently on multiple tablets for flexible placement | Physically positions screens for required sightlines |
| 22mm character height | Enforces 22mm minimum with screen calibration tool | Completes screen calibration during setup using bank card method |
| No all-caps text | Enforces mixed case throughout | None (handled entirely by software) |
| Text-background contrast | Uses high-contrast colour schemes | None (handled entirely by software) |
| Alert tones before required announcements | Plays audible chime before termination, diversion, and hail-and-ride announcements | None (handled entirely by software) |

---

## Compliance Notes

### Note 1: Alert — Audio and Visual Components (Implementation Cross-Cutting Requirement)
The alert requirement spans regulations 8(2), 10(2)(b), 11(2)(b), and 11(5)(b). It is implemented as two co-equal features: FR-AT-27 (audio chime) and FR-AT-65 (500ms visual screen flash), both firing simultaneously before the announcement overlay text. The same audio chime file is used across all four alert-required announcements for consistency. All announcement audio following the chime is pre-rendered (FR-AT-28). This is an inviolable architectural rule and is documented in CLAUDE.md as such — any future change to announcement code must preserve both the alert-chime-before-audio-file sequence and the simultaneous visual flash for these four announcement types.

### Note 2: Adaptive Volume Control — Not in Scope
The PDS does not implement adaptive volume control (Reg 13(3)). This means the operator must use pre-measured ambient volume (Reg 13(2)) rather than live-measured. This is fully compliant — the regulations allow either approach. Operators should be informed that adaptive volume is not included and they must set volume manually.

### Note 3: Audio Induction Loop — Operator Hardware
The PDS outputs audio through standard Bluetooth or wired connections (Reg 13(5)). Hearing loop compatibility depends entirely on the operator's hardware setup. The software's audio output is compatible with induction loops — it simply needs to be connected. The operator must be clearly informed of this requirement.

### Note 4: Rearward-Facing Wheelchair Spaces — Reg 14(3)(b)
For vehicles first used on or after 1 October 2024, displays must be visible from all wheelchair spaces including rearward-facing ones. This typically requires a forward-facing screen at the rear of the bus. The PDS supports this by allowing operators to deploy additional tablets running the same software; each tablet pairs independently and runs its own journey state, but because they're tracking the same physical bus, their journey progress is approximately synchronised. True primary-secondary tablet linking with shared journey state is a future-release feature.

### Note 5: Welsh Language — Known Omission
The Accessible Information Regulations apply to England, Wales, and Scotland. Services operating in Wales may have additional Welsh language requirements. The PDS is English-only. Welsh language support is listed as a future-release feature in the PRD and must be addressed before commercial deployment on Welsh routes.

### Note 6: GMS Certification — Device Requirement
The PRD requires tablets to be Google Mobile Services (GMS) certified to ensure FusedLocationProviderClient and FCM function correctly. Non-GMS tablets would compromise GPS accuracy and sync reliability, potentially affecting compliance with Regulation 9 (timely next-stop announcements).

### Note 7: Content Accuracy is the Operator's Responsibility
The software displays the data entered into the dashboard. If a fleet manager enters an incorrect stop name, a missing stop, or the wrong final destination, the regulation-required announcements will be technically delivered but factually wrong. The operator is responsible for the accuracy of the content they author. The dashboard provides search via the authoritative NaPTAN dataset to minimise data-entry errors, but final content responsibility rests with the operator.
