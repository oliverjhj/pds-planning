# CONTEXT.md
# Passenger Display System (PDS) — Living Context

**Version:** v3.7
**Last Updated:** May 2026

This document is the architect's living memory across chats. It is the only knowledge file that changes during the project's lifetime; the others (PRD, Data-Architecture, Compliance, WORKFLOW) are frozen.

Re-uploaded to the Claude Project whenever it's updated. Read at the start of every new architect chat.

---

## Orientation

The Passenger Display System (PDS) is a two-surface SaaS product providing legally compliant audio and visual passenger information on UK buses operating rail replacement services. The product consists of an Android tablet application (kiosk-mode, offline-first, GPS-driven, regulation-compliant) and a Next.js web dashboard where bus operators author routes and manage their fleet. Both share a single Supabase backend.

An earlier build attempt at this product was abandoned due to scope drift and architectural disconnects between planning and implementation. This project is a clean restart with revised planning documents and tighter workflow conventions (see WORKFLOW.md).

---

## 1. Current State

### What Exists

- Five planning documents finalised at v3.7 following a seven-task post-adversarial-review re-planning campaign:
  - PRD.md v3.7
  - Data-Architecture.md v3.7
  - Compliance-Mapping-Matrix.md v3.7
  - WORKFLOW.md v3.7
  - CLAUDE.md (Android) and CLAUDE.md (Dashboard) — two separate files, one per repository
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
17. Audio architecture: pre-rendered MP3 files, rendered server-side at route-save time by the `render-route-audio` Edge Function, distributed to tablets via sync. On-device TTS removed entirely. Fixed announcement texts (termination, H&R start/end, diversion start/end, alert chime) bundled in APK using same TTS voice for consistency. Diversion audio is a generic fixed phrase; specific affected stop names are conveyed visually via tube-map strikethrough. (Task 3)
18. Device status enum: `status TEXT CHECK IN ('pending', 'active', 'suspended')` replaces `is_approved BOOLEAN`. Dashboard and tablet UI show distinct messages per state; suspended and pending are never conflated. (Task 4)
19. Device secret for recovery: 256-bit cryptographically random secret generated at pairing, returned to tablet once in the pairing response, stored as SHA-256 hash server-side. `recover-device` requires both Android ID (row lookup key) and device secret (authentication). Android ID alone is not a credential. (Task 4)
20. FCM promoted to Must Have. Three-trigger sync model formalised: ConnectivityManager (offline→online), FCM push notification (responsive, sub-5-minute propagation), 30-min WorkManager periodic (safety net). FCM is the primary mechanism for timely route propagation to online tablets. (Task 5)
21. Heartbeat mechanism: 2-minute `last_seen_at` update, independent of route sync. Journey-path: Handler loop inside foreground GPS service (reliable, protected from OEM battery kills). Idle-path: WorkManager PeriodicWorkRequest (best-effort; not guaranteed on hostile-OEM hardware — documented acceptable gap). Fleet managers should treat idle tablets as "not in service." (Task 5)
22. Kiosk Level 2 (Device Owner mode / Android Lock Task Mode) deferred to Could Have. Level 1 soft kiosk (screen pinning + default launcher registration) is the only kiosk mode in the initial release. Architecture accommodates Level 2 without redesign; only provisioning differs. (Task 6)
23. Tablet NaPTAN bundle cut. Tablet holds no NaPTAN data. Stop names, coordinates, and NaPTAN IDs travel with routes via `route_stops` snapshot copied at dashboard route-creation time. NaPTAN is dashboard-only. (Task 6)
24. Upload-sync scaffolding cut. Sync is download-only in the initial release. The "check → upload → download" sequence and all upload-phase language removed from PRD, Data-Architecture, and WORKFLOW. (Task 6)
25. Visual alert defined (FR-AT-65): 500ms high-contrast screen flash fires simultaneously with the audio chime, before the announcement overlay text appears. Same four trigger types as the audio chime (termination, diversion start, H&R start, H&R end). Together with FR-AT-27 (audio chime), constitutes the combined alert required by Regs 8(2), 10(2)(b), 11(2)(b), 11(5)(b). (Task 7)
26. Multi-tablet audio designation: `audio_enabled BOOLEAN` per device in the `devices` table, configurable from the dashboard fleet view. First tablet paired to an operator defaults to `true`; subsequent tablets default to `false`. Tablets with `audio_enabled = false` suppress all audio output but continue full visual display. Prevents overlapping audio from multiple tablets on the same bus. (Task 7)
27. `device_pairing_codes` primary key changed from `code TEXT` to `id UUID DEFAULT gen_random_uuid()`. `code` demoted to a regular UNIQUE indexed column. Decouples row identity from the code value, avoiding entanglement between cleanup, audit retention, and collision checking. (Task 7)
28. `devices.active_route_id` FK changed to `ON DELETE SET NULL`. Ensures route hard-deletion (possible after 90-day retention period per Data-Architecture §10) does not cascade to or block device rows. (Task 7)
29. Return route divergence detection: `routes.last_synced_with_return TIMESTAMPTZ NULLABLE` set on both routes when a return is generated. Dashboard warns operator when editing a route with a linked return whose `updated_at > last_synced_with_return`; offers [Re-generate return] or [Keep existing]. Warning-and-regenerate only, not a diff UI. (Task 7)
30. Room migration policy added to WORKFLOW.md §10.3: every Room schema change must include a Migration object or an explicit `fallbackToDestructiveMigration()` with rationale; migration must be tested on a real device with pre-existing data before commit. Prevents the schema-mismatch crash class that terminated the previous build. (Task 7)

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

After the initial five planning documents were drafted, an adversarial review identified structural weaknesses across seven areas: stop-detection model, online/offline and sync story, device identity and auth, `is_approved` boolean overloading, compliance gaps (driver panel occlusion, multi-tablet audio, missing visual alert, calibration verification gap), features that look small but aren't (Kiosk Level 2, NaPTAN tablet bundle, upload-sync scaffolding), and frozen-document contradictions (WORKFLOW vs CONTEXT on repo count, no Room migration policy). A seven-task re-planning campaign addressed each area in sequence: Task 1 (stop detection), Task 2 (hail-and-ride and diversion model), Task 3 (pre-rendered audio architecture), Task 4 (auth, device identity, status enum), Task 5 (FCM, heartbeat, online status), Task 6 (scope cuts), and Task 7 (compliance gaps, WORKFLOW fixes, smaller schema items, parked contradictions, campaign close-out). All five documents are now at v3.7. The frozen-documents rule re-engages; no further edits without a fresh re-planning trigger. Next: Stage 1 — web dashboard MVP.

---

## 4. Reference Notes

Cross-cutting facts that don't fit cleanly in the sections above but matter for future tasks.

- **Two CLAUDE.md files exist.** The Android repo's CLAUDE.md covers Kotlin/Hilt/Room/GPS conventions; the dashboard repo's CLAUDE.md covers Next.js/TypeScript/Supabase conventions. References in PRD compliance sections (alert chime FR-AT-27, visual alert FR-AT-65, audio rules FR-AT-28) refer to the Android CLAUDE.md.
- **One Supabase project, shared schema.** The dashboard and the Android app use the same Supabase project, the same tables, the same RLS policies. Any schema change affects both surfaces. The architect coordinates schema changes deliberately.
- **NaPTAN is dashboard-only.** The `naptan_stations` table lives in Supabase and is queried only by the dashboard route builder. Tablets hold no NaPTAN data; stop names, coordinates, and NaPTAN IDs travel with routes via `route_stops` (copied at route-creation time on the dashboard).
- **The frozen documents are read-only for the build.** All five documents are at v3.7. If during the build the architect believes a frozen document is wrong, that triggers re-planning, not editing. See WORKFLOW.md section 13.
- **Architect chats can stay long.** Unlike claude-code chats, this Anthropic project's chats hold long planning sessions without significant degradation. New architect chats start when the current one feels muddled or hits compression.
- **The pilot customer is a personal contact's bus company.** This is helpful for real-world testing but should not be the sole basis for declaring product-market fit. Plan to have at least one paying customer outside the immediate network before generalising.
