# CLAUDE.md
# Passenger Display System (PDS) — Web Dashboard

**Version:** 3.9 (May 2026)

This file is your reference for working on the PDS web dashboard. Read it at the start of every session. It contains conventions, architectural rules, and known gotchas. If a rule here conflicts with a prompt, ask before deviating.

## Changelog

### v3.9 (May 2026)
Round-3 post-adversarial-review close-out sweep. Applies the per-prompt CLAUDE.md impact lists produced by round-3 Tasks 2 and 3. Substantive deltas — itemised in the v3.9 sweep summary at the end of this file — include: audio file format switched MP3 → **LINEAR16 WAV** end-to-end (Rule 10 narrative, "LINEAR16/WAV end-to-end" gotcha); **pg_boss configuration inviolable rule** (`{ supervise: false, schedule: false }` + hourly `pgboss-maintain` scheduled function) added; **`audio-render-worker` IS a scheduled Supabase Edge Function** (corrects the v3.8 Setup Notes wording that called it a separately-deployed runtime) — verified empirically in `spike-records/round-3/findings-pg_boss.md`; FR-WD-21 two-layer deduplication gotcha; FR-WD-19 cross-reference fixed (now §FR-WD-22 + §5.2 for the audio-toggle context — FR-WD-19 was renumbered to HTTPS in round 2); **staging environment** setup added (two Supabase projects + two Vercel targets + six Sentry DSNs); **operator-practice note** linking PRD §11.2 item 9 to the FR-WD-13 global failed-render indicator surface; Reg 13(4) frequency-range now empirically verified.

### v3.8 (May 2026)
Previously unversioned. Brought into sync with the v3.8 planning suite via the round-2 close-out sweep. Reflects all round-1 (PRD/Data-Arch/Compliance v3.0 → v3.7) and round-2 (v3.7 → v3.8) decisions that had never propagated into CLAUDE.md. Substantive deltas — itemised in the v3.8 sweep summary at the end of this file — include: operator-status three-state enum (`pending`/`active`/`suspended`) replacing boolean `is_approved`, with `/suspended` route added; audio pipeline replaced by `pg_boss` job queue + Google Cloud TTS + version-keyed Storage paths + render-then-FCM ordering (dashboard now calls `enqueue-render-job`, never the deprecated synchronous `render-route-audio` Edge Function); per-device `audio_enabled` toggle in fleet view with honest one-per-bus framing; FR-WD-12 divergence detection rewritten to structural `stops_content_hash` comparison; `devices.activation_state` rename; `routes.audio_render_status` + `routes.audio_version` + `routes.audio_announcement_hash` + `routes.stops_content_hash` columns; `journey_summaries` + `rate_limit_attempts` shared tables; `route-audio` Storage bucket; manual custom-access-token-hook setup step; Sentry crash telemetry; hard-delete language removed; anonymous Auth user accumulation acknowledged.

---

## What This Project Is

A Next.js web dashboard where UK bus operators register their company, manage their fleet of Android tablets, and author the routes that those tablets display. It is the cloud-side companion to a separate Android application (different repository).

Bus operators sign up via this dashboard, get manually approved by the system administrator, then use the dashboard to create routes, generate device pairing codes, and monitor their fleet.

The product is legally regulated under the UK Public Service Vehicles (Accessible Information) Regulations 2023. The dashboard contributes to compliance indirectly: the routes authored here are what tablets display to passengers, so accuracy matters. The dashboard also kicks off the server-side audio render pipeline that produces the regulated pre-rendered announcements; render-then-FCM ordering is mandatory (see architectural Rule 10).

---

## Tech Stack

- **Framework:** Next.js 15+ with App Router (not Pages Router)
- **Language:** TypeScript (strict mode, no `any` without justification)
- **Hosting:** Vercel (free tier)
- **Backend:** Supabase (shared with the Android app — same project, same tables). JWTs carry `operator_id` as a custom claim stamped by a server-side custom-access-token hook (manual Supabase Auth console setup — see "Setup Notes").
- **Auth:** Supabase Auth (email + password)
- **Data fetching:** Server Components and Server Actions; minimal client-side Supabase SDK use for interactive bits
- **Styling:** Tailwind CSS
- **Components:** shadcn/ui (Radix UI primitives + Tailwind)
- **Forms:** React Hook Form + Zod for validation
- **Icons:** Lucide React (bundled with shadcn/ui)
- **Crash telemetry:** Sentry (Next.js SDK; both server and edge runtimes). See PRD §NFR-R-07.

---

## Reference Documents — DO NOT READ

The `/docs` folder in this repo contains four reference documents:

- `/docs/PRD.md` — product requirements
- `/docs/Data-Architecture.md` — database schemas and sync algorithm
- `/docs/Compliance-Mapping-Matrix.md` — regulatory compliance evidence
- `/docs/WORKFLOW.md` — how the project is built

**Do not read these files in normal task work.** They are for the project's architect, not for you. They are also **frozen** — they were finalised at planning time and do not change during the build. Never edit them.

If you think you need information from those documents to complete a task, ask the user. The architect will either include the relevant excerpt in the prompt or add a rule to this CLAUDE.md.

---

## Available MCP Tools

Three MCP servers are configured at the user scope and available in every session:

- **Context7** — fetches current official documentation for libraries on demand. Use it freely before writing code that uses a library where the API may have changed since training. The dashboard's stack moves fast — particularly Next.js (App Router patterns and server actions have evolved significantly), the Supabase JS SDK, shadcn/ui, and React Hook Form. A quick Context7 lookup catches outdated API patterns before they hit runtime.

- **GitHub** — read-only by convention. You can read repo state, list issues, read PR content. You do **not** push to GitHub, you do **not** open PRs, you do **not** modify repo settings — these are manual operations by the system administrator regardless of what the MCP technically permits.

- **Supabase** — your most-used MCP for dashboard work, and the most sensitive. It runs in `--read-only` mode by default. In read-only mode you can: read schema, query tables, list Edge Functions, inspect storage, read RLS policies. You **cannot** run migrations, modify data, deploy Edge Functions, or change RLS policies unless the system administrator has explicitly reconfigured the MCP for the current task. The architect's prompt will state when write access has been granted for a task. After any write task, the MCP returns to read-only — do not assume write access persists between prompts.

  Common dashboard tasks that require Supabase MCP write access (and so require explicit architect authorisation): creating or modifying tables, writing or updating RLS policies, deploying Edge Functions, creating Storage buckets. Dashboard tasks that are fine in read-only mode: querying data for development, inspecting schema to write correctly-typed client code, generating TypeScript types from the schema.

A fourth MCP, **Playwright**, will be installed before Stage 1 dashboard verification (end-to-end browser tests of signup, route creation, device pairing). It is not yet available; do not attempt to invoke it.

When in doubt about whether an MCP-driven operation is authorised for the current task, ask before running it.

---

## Project Structure

The target structure is below. Create directories as needed; do not pre-create empty directories.

    src/
    ├── app/                          # Next.js App Router
    │   ├── (auth)/                   # Routes with auth layout (login, signup, password reset)
    │   │   ├── login/
    │   │   ├── signup/
    │   │   └── layout.tsx            # Auth layout (centred form, no nav)
    │   ├── (dashboard)/              # Routes with dashboard layout (logged-in users)
    │   │   ├── routes/               # Route list, route detail, route create/edit
    │   │   ├── devices/              # Device fleet view, pairing, audio_enabled toggle
    │   │   ├── account/              # Account settings
    │   │   ├── pending/              # "Pending approval" landing (status = 'pending')
    │   │   ├── suspended/            # "Account suspended" landing (status = 'suspended')
    │   │   └── layout.tsx            # Dashboard layout (nav, header)
    │   ├── api/                      # Route handlers if needed (rare — prefer server actions)
    │   ├── layout.tsx                # Root layout
    │   └── page.tsx                  # Landing / redirect to login or dashboard
    ├── components/
    │   ├── ui/                       # shadcn/ui components (generated, don't hand-edit)
    │   ├── routes/                   # Route-specific components (RouteBuilder, StopList, etc.)
    │   ├── devices/                  # Device-specific components
    │   └── shared/                   # Shared components (Header, Nav, etc.)
    ├── lib/
    │   ├── supabase/                 # Supabase client setup (server, client, middleware variants)
    │   ├── actions/                  # Server actions, organised by domain
    │   │   ├── routes.ts             # createRoute, updateRoute, deleteRoute
    │   │   ├── devices.ts            # generatePairingCode, deactivateDevice, renameDevice, setAudioEnabled
    │   │   └── account.ts            # updateProfile, changePassword
    │   ├── schemas/                  # Zod schemas for validation
    │   ├── types/                    # Shared TypeScript types (often generated from Supabase)
    │   └── utils/                    # Generic utilities (formatDate, cn, etc.)
    ├── sentry.client.config.ts       # Sentry browser init
    ├── sentry.server.config.ts       # Sentry server runtime init
    ├── sentry.edge.config.ts         # Sentry edge runtime init
    └── middleware.ts                 # Auth middleware (refresh session on every request)

    docs/                             # Frozen reference documents (DO NOT READ)
    ├── PRD.md
    ├── Data-Architecture.md
    ├── Compliance-Mapping-Matrix.md
    └── WORKFLOW.md

---

## Build, Run, and Test Commands

    npm run dev                       # Start dev server (localhost:3000)
    npm run build                     # Production build
    npm run start                     # Run production build locally
    npm run lint                      # ESLint
    npm run typecheck                 # tsc --noEmit
    npm run test                      # Vitest (when set up)

    npx supabase gen types typescript --project-id <id> > src/lib/types/database.ts
                                      # Regenerate Supabase TypeScript types from schema
                                      # Reading current schema for this can also go via the
                                      # Supabase MCP (read-only) rather than the CLI.

Deployment to Vercel happens automatically on push to `main`. The user manages Vercel deployment manually; you don't trigger deploys.

---

## Setup Notes

Manual setup steps that are easy to miss and have non-obvious failure modes if forgotten.

- **Custom access token hook (Supabase Auth console — manual step).** Server-side hook that stamps `operator_id` into the JWT custom claims for every issued token. This is a manual action in the Supabase Auth dashboard; it does NOT travel in migrations. Without it, **every operator-scoped RLS policy quietly returns empty** for both dashboard and tablet sessions. See Data-Architecture §12 setup checklist. After any project reset, re-register before debugging anything else.
- **Sentry DSN.** Configured via Vercel environment variable; do not commit the DSN to git. **Stage 1 setup uses six DSNs total** — three surfaces (Android, dashboard, Edge Functions) × two environments (staging, production). See "Staging environment" below.
- **pg_boss schema.** The audio pipeline runs on `pg_boss` (Postgres-backed job queue). `pg_boss` owns its own schema in the Supabase database — see Data-Arch §11. Migrations are managed by `pg_boss` itself, not by the dashboard repo.
- **`audio-render-worker` is a scheduled Supabase Edge Function** (round-3 verification spike correction). The worker polls the `pg_boss` queue on a schedule from inside a Deno Edge Function isolate. **It MUST construct pg_boss with `{ supervise: false, schedule: false }`** — the default configuration assumes a long-lived process and fights the Deno isolate runtime. A separate scheduled Edge Function `pgboss-maintain` calls `boss.maintain()` hourly (cron `0 * * * *`) to handle archival, retention, and expiry — those operations live in the supervisor that the worker disables. The configuration is verified empirically in `spike-records/round-3/findings-pg_boss.md` and is treated as an inviolable architectural rule (also stated in Rule 10).
- **Staging environment.** Stage 1 setup requires **two Supabase projects** (production and staging), **two Vercel deployment targets**, and **six Sentry DSNs** (three surfaces × two environments). Vercel preview deploys point at staging; `main`-branch deploys point at production. The full setup checklist, including ordering dependencies between the manual steps (Supabase project create → custom access token hook → schema migrations → Edge Function deploy → Sentry DSN configuration → Vercel env vars), is in Data-Arch §12. Do not skip the staging environment for "speed" — without it, the only way to verify a schema change is against production.

---

## Architectural Rules — DO NOT VIOLATE

These are the rules whose violation breaks the architecture. Treat them as inviolable. Bracketed tags categorise each rule for quick scanning; rule numbers are stable references (e.g., "Rule 9").

### 1. [Next.js] Server Components by default, client components only when needed

Next.js App Router is Server Component-first. Every component you write defaults to running on the server. Only add `"use client"` when you genuinely need client-side interactivity (forms with complex state, drag-and-drop, charts, anything using browser APIs).

If you find yourself adding `"use client"` to the top of every file, stop and reconsider. Most of the dashboard is data display, which is a server-component case.

### 2. [Next.js] Mutations are Server Actions, not API routes

For any operation that changes data (create/update/delete route, generate pairing code, toggle `audio_enabled`, etc.), use a Server Action. Do not create a `/app/api/` route handler unless you have a specific reason (e.g., a third-party webhook).

Server Actions live in `src/lib/actions/`. They are exported async functions marked with `"use server"`. They are called directly from server components (via form actions) or from client components (via `useTransition` or form actions).

Example structure:

    // src/lib/actions/routes.ts
    "use server";

    import { revalidatePath } from "next/cache";
    import { createServerSupabaseClient } from "@/lib/supabase/server";
    import { RouteSchema } from "@/lib/schemas/route";

    export async function createRoute(formData: FormData) {
      const supabase = await createServerSupabaseClient();
      const parsed = RouteSchema.parse(Object.fromEntries(formData));
      // ... insert into Supabase via the replace_route_with_stops RPC (Rule 9)
      revalidatePath("/routes");
    }

### 3. [Auth] Auth checks at the layout level, not per-component

Every protected route is under `(dashboard)/layout.tsx`. That layout fetches the user once via `getUser()` and either renders children or redirects. Individual pages do not re-check auth.

The middleware (`src/middleware.ts`) refreshes the Supabase session on every request to keep tokens fresh. The layout does the actual gating.

### 4. [Auth + Gate] The operator-status gate is a three-state enum, server-enforced

After auth, every dashboard action must check the operator's `status` column. Values: `pending`, `active`, `suspended`. **Distinct UX per state** (PRD §FR-WD-04, §FR-WD-05; CONTEXT Decision #18):

- **`pending`** → redirect to `/pending`. Show the "account pending approval" page. Account-profile pages remain accessible so the operator can update contact details; everything else is gated.
- **`active`** → normal dashboard access.
- **`suspended`** → redirect to `/suspended`. Show the "account suspended — contact support" page. **Read-only** access to historical data; no route edits, no device pairing, no fleet mutations. On the tablet side, in-flight journeys continue to natural end — operator-suspension is honoured at journey end, not mid-journey (see Android CLAUDE.md Rule 15-equivalent decisions).

Enforcement is layered: **RLS at the database** (mutation queries silently return empty for non-`active` operators) AND the dashboard's `(dashboard)/layout.tsx` performs the redirect. Do not rely on either layer alone. Do not implement "soft" approval gates in individual components — the single chokepoint is the dashboard layout (plus RLS).

The old `is_approved BOOLEAN` is gone. Anywhere you find code referencing it, it is wrong.

### 5. [Auth] Supabase queries respect RLS — do not bypass it

The dashboard always uses the user-scoped Supabase client (with the user's session JWT, which carries `operator_id` via the custom-access-token hook). Never use the service role key from the client or from regular server actions. RLS policies are the authoritative security boundary.

The service role key is used only by Edge Functions and by the `audio-render-worker` (both of which run outside Next.js). If a dashboard operation seems to require service role privileges, that operation belongs in an Edge Function, not in a Server Action.

Examples requiring Edge Functions, not Server Actions:
- Generating pairing codes (writes to `device_pairing_codes` are service-role-only)
- Creating anonymous Supabase users (admin API)
- Modifying device JWTs
- Enqueuing audio render jobs (`enqueue-render-job` — Rule 10)

The dashboard calls Edge Functions via `supabase.functions.invoke()`.

### 6. [Routes] Routes are authored on the dashboard, not the tablet

The tablet is read-only for routes. The dashboard is the **only** authoring surface. All CRUD on routes happens here.

This means the dashboard's route builder is the most feature-rich UI in the product. It must handle: NaPTAN search, drag-and-drop stop reordering, return route generation, per-stop `proximity_radius_meters` configuration, per-stop `segment_type` (`scheduled` vs `hail_and_ride`), route metadata, validation, error states. Build this carefully.

### 7. [NaPTAN] NaPTAN data is queried, not bundled

The dashboard queries NaPTAN data from the Supabase `naptan_stations` table via Postgres full-text search. Do not download the entire NaPTAN dataset to the client. The table has ~400,000 rows; only return what matches the user's search query.

Search is debounced (250ms typical). Results limited to ~20 hits per query.

### 8. [Routes] Stop data is copied into `route_stops` at creation

When the user adds a stop to a route, the NaPTAN ID, name, CRS code, latitude, and longitude are *copied* into the new `route_stops` row. The `route_stops` table does not foreign-key to `naptan_stations`. This means routes survive NaPTAN data changes (a station rename in NaPTAN does not affect saved routes).

This is the same rule as the Android side. The behaviour must be consistent across both surfaces — and is structurally necessary, since the tablet holds no NaPTAN database to reference.

### 9. [Routes] Use the atomic route + stops RPC for saves

When saving a route (create or update), call the `replace_route_with_stops` Supabase RPC. This is a single transactional operation that upserts the route, replaces all its stops, **and computes `routes.stops_content_hash`** from the just-inserted stops (canonical form per Data-Arch §4.4).

Do not:
- Insert the route and then insert stops in separate queries (race condition).
- Delete stops before inserting new ones in separate queries (orphans stops if insert fails).
- Compute `stops_content_hash` client-side. The hash is the integrity contract for FR-WD-12 divergence detection; it must be authoritative and is computed by the RPC.

After the RPC succeeds, the Server Action enqueues an audio render job via the `enqueue-render-job` Edge Function (Rule 10). The save endpoint does not wait for render completion — it returns immediately and the UI shows "Audio rendering" until `routes.audio_render_status = 'ok'`.

### 10. [Audio] Audio rendering happens via `pg_boss`, not by the dashboard

The dashboard **enqueues** an audio render job and returns. The `audio-render-worker` Edge Function — a scheduled Supabase Edge Function — consumes the `render-route-audio` job from the `pg_boss` queue, calls **Google Cloud TTS** (voice `en-GB-Neural2-B`, locked, `audioEncoding: 'LINEAR16'`) for each line, writes versioned **WAV files (LINEAR16 PCM, 24 kHz mono)** to the `route-audio` Storage bucket at `{operator_id}/{route_id}/{route_version}/...`, and on success bumps `routes.audio_version`-equivalent state (`audio_render_status = 'ok'`, `audio_announcement_hash` updated). The bucket retains the latest **three** versions per route (round-3 increase from two — supports an in-flight tablet, the new active version, and one rollback target).

**pg_boss configuration (INVIOLABLE RULE).** The `audio-render-worker` MUST construct pg_boss with `{ supervise: false, schedule: false }`. The default pg_boss configuration assumes a long-lived host process and starts background maintenance loops that conflict with the Deno Edge Function isolate's short, scheduled invocations. A separate scheduled Edge Function `pgboss-maintain` calls `boss.maintain()` hourly (cron `0 * * * *`) to handle the archival/retention/expiry work that the disabled supervisor would otherwise perform. This configuration is **verified empirically** in `spike-records/round-3/findings-pg_boss.md`; the previous v3.8 framing (that the worker was a "separately-deployed runtime, not an Edge Function") was incorrect and has been corrected throughout v3.9. See Data-Arch §4.6 (pg_boss INVIOLABLE RULE subsection).

**Render-then-FCM is mandatory.** The worker — not a DB trigger, not the dashboard — dispatches the FCM data-only push that triggers the tablet to sync, AFTER the audio files are in Storage. Tablets must not be told to sync before the new audio is available, or they will 404 trying to download it.

**Differential re-render.** If `audio_announcement_hash` is unchanged for the route-announcement text, the worker skips synthesis for that file and copies the file from the previous `route_version` directory to the new one. Same for per-stop files keyed on the stop name (Data-Arch §4.6).

**The deprecated synchronous `render-route-audio` Edge Function is gone.** Do not call it; it does not exist. The name `render-route-audio` survives only as the `pg_boss` **job name** queued by `enqueue-render-job`.

**Reg 13(4) frequency-range presence has been empirically verified** on the LINEAR16 output (see Compliance Mapping Matrix and `spike-records/round-3/findings-tts-frequency.md`). The previous "TTS verification required pre-deployment" / "BLOCKING WORK" framing no longer applies. A future voice change is itself a Case 1 re-planning event and would require fresh verification.

### 11. [Audio] `audio_enabled` toggle in the fleet view

The fleet view exposes a per-device `audio_enabled` boolean toggle (Server Action: `setAudioEnabled`). Defaults: the first tablet paired to an operator → `true`; subsequent tablets → `false`.

**Honest framing.** This is a **per-device** control. The operator's responsibility is to enable audio on exactly one tablet per bus. The dashboard surfaces a **non-blocking warning** when more than one tablet in an operator's fleet has `audio_enabled = true` (e.g., "3 tablets have audio enabled — ensure only one per bus") — and a **separate** non-blocking warning (FR-WD-22) when an operator's fleet has zero `audio_enabled = true` devices. The dashboard does **not** enforce the one-per-bus constraint because the system cannot tell which physical bus a tablet is on. See PRD §FR-WD-22 and §5.2 (FR-WD-19 was renumbered to HTTPS in round 2 and no longer relates to the audio toggle).

Tablets with `audio_enabled = false` skip audio downloads entirely (not just playback) — see Android CLAUDE.md Rule 12. Toggling a tablet from `false` to `true` will trigger an audio download on the tablet's next sync.

**Operator practice — route edits before service.** PRD §11.2 item 9 names the operator-side practice that pairs with the FR-AT-28 driver hint on the tablet: fleet managers should verify `routes.audio_render_status = 'ok'` for affected routes before the next service, particularly for routes edited shortly before that service runs. The dashboard's **route-list view** (FR-WD-13) and the **global failed-render indicator** (FR-WD-13, persistent across pages, dismissible per-route only) are the primary surfaces fleet managers use; both should be implemented to be visible and obvious. A failed render that an operator does not notice produces the "compound fleet-blocked window" risk row in PRD §13 — minimise it by making the surface impossible to miss.

### 12. [Routes] FR-WD-12 divergence detection is structural (content hash)

When editing a route whose `linked_return_route_id` is set, compare the source route's `stops_content_hash` to the linked return route's `stops_content_hash_at_generation`. If they differ, surface a divergence warning with options `[Re-generate return]` / `[Keep existing]`.

**Do not use timestamps.** The old `last_synced_with_return TIMESTAMPTZ` mechanism is deprecated — it fired on every edit (warning fatigue). The structural hash comparison fires only when the stop list actually changes. `last_synced_with_return` is retained as a soft audit signal but is no longer the divergence trigger. See PRD §FR-WD-12 and Data-Arch §2.4.

### 13. [Time] All timestamps in UTC

Supabase stores `TIMESTAMPTZ` in UTC. When displaying timestamps to the user, convert to UK local time (handles GMT/BST) using `Intl.DateTimeFormat` with `timeZone: 'Europe/London'`. Never use raw `Date` formatting that depends on the server's locale.

### 14. [Forms] Forms validated with Zod

Every form has a Zod schema in `src/lib/schemas/`. Server Actions parse `FormData` through that schema before touching Supabase. Client-side validation (via React Hook Form's Zod resolver) provides immediate feedback; server-side validation is the authoritative check.

Schemas are shared between client and server. Do not duplicate validation logic.

### 15. [TypeScript] TypeScript strict mode

The project is in TypeScript strict mode. `any` is banned without a written justification comment. Supabase types are generated from the schema using `supabase gen types` — use the generated types, do not write your own.

When the generated types are stale (e.g., a migration added a column), regenerate them. Don't paper over the gap with `as any` casts.

---

## Key Data Decisions

**One Supabase project, two surfaces.** The dashboard and the Android app share the same Supabase project, the same tables, the same RLS policies. Schema changes affect both. Coordinate via the architect.

**Email/password auth for operators.** No magic links, no social login, no MFA in the initial release. Supabase Auth handles the flow (signup, email verification, password reset). Use the default Supabase Auth UI flows or build matching screens.

**Anonymous users for devices.** Tablets pair via Edge Functions that create anonymous Supabase Auth users. The dashboard never creates these users; it only triggers the flow by generating pairing codes. Devices are linked to operators via the `devices` table's `operator_id` and `user_id` columns. **Note:** anonymous users accumulate over time (no automatic cleanup in the initial release) — see Known Gotchas.

**Operator → user mapping is one-to-one.** Each operator has exactly one Supabase Auth user. No team accounts. If multiple people at a bus company need access, they share the login.

**Operator status is a three-state enum, not a boolean.** `operators.status TEXT CHECK IN ('pending', 'active', 'suspended')`. Replaces the old `is_approved BOOLEAN`. See Rule 4.

**Device activation state is a separate enum.** `devices.activation_state TEXT` (renamed from `devices.status` in round-2 Task 3 to disambiguate from `operators.status`). Values per Data-Arch §2.2. The `pair-device` and `recover-device` Edge Functions read this.

**Per-device `audio_enabled` boolean** in `devices`. Defaults: first tablet paired to operator → `true`; subsequent → `false`. See Rule 11.

**Route-level audio columns** on `routes`:
- `audio_version` (epoch millis; equals `updated_at`) — used as the Storage path segment.
- `audio_render_status` enum (`pending` / `ok` / `failed`) — surfaced in the route list; used as the tablet's journey-start gate.
- `audio_render_error` — set when `audio_render_status = 'failed'`; surfaced for diagnostics.
- `audio_announcement_hash` — SHA-256 of the route-announcement text; drives differential re-render.
- `stops_content_hash` — SHA-256 of the canonical stop-list serialisation; drives FR-WD-12 (Rule 12).

**Soft handling of route deletion.** Deleting a route sets `is_deleted = true`, not a hard `DELETE`. Tombstone propagates to tablets via sync. **Routes are not hard-deleted on a fixed schedule in the initial release** — the old 90-day hard-delete reference was removed in round-2 Task 3. If a hard-delete is ever introduced later, `devices.active_route_id` is already `ON DELETE SET NULL` to prevent cascade-blocks.

**NaPTAN search via Postgres full-text.** The `naptan_stations.search_vector` column is a generated `TSVECTOR`. Queries use `search_vector @@ websearch_to_tsquery('english', :query)`. Results ranked by `ts_rank`. Return the top 20.

**`journey_summaries` table** (Supabase). Anonymous count metrics uploaded by tablets at journey end. RLS scoped by `operator_id`. The dashboard surfaces a paginated list per fleet for diagnostics. See PRD §FR-AT-66.

**`rate_limit_attempts` table.** Server-side rate-limit storage for `pair-device` (per-IP) and `recover-device` (per-Android-ID). 24-hour retention via the daily 03:00 UTC cleanup. Service-role only — the dashboard does not read or write it. See Data-Arch §2.9.

---

## Workflow Rules

### Plan before building

Enter plan mode for any task touching 3+ files or involving architectural decisions. Produce a plan, wait for user confirmation, then execute. Do not start changes during plan mode.

### One feature at a time

Complete, test, and commit each feature before starting the next. Reference the PRD requirement being implemented in the commit message (e.g., "feat: implement FR-WD-08 route creation flow"). The user provides the PRD requirement number in the prompt.

### Verify before marking done

Run `npm run build` and `npm run typecheck` before claiming the feature works. Build failures and type errors are not "edge cases" — they're broken code. The user verifies the feature behaviour in the browser; you verify it compiles cleanly.

### Keep it simple

Make every change as simple as possible. Touch minimal code. Find root causes, not temporary fixes. If a fix feels hacky, pause and find the clean solution — or ask.

### Do not refactor opportunistically

If you notice unrelated code that "could be improved," leave it. The current task's scope is defined by the prompt. Drive-by refactoring causes scope drift and was a recurring problem in the previous (Android) build attempt.

### Commit messages

The user provides the commit message at the end of each task. Use the format:

    <type>: <short title referencing the PRD requirement>

    <detailed description in 2-4 paragraphs covering:
     - what was changed and why
     - which PRD requirements this implements
     - any notable decisions or trade-offs>

`<type>` is one of: `feat`, `fix`, `refactor`, `docs`, `chore`, `test`.

---

## Known Gotchas

These are concrete problems that often bite Next.js + Supabase dashboards, plus round-2 additions for the v3.8 architecture.

### Server vs client Supabase clients are different

Three variants exist in `src/lib/supabase/`:

- `server.ts` — for Server Components and Server Actions. Reads cookies via Next.js `cookies()`.
- `client.ts` — for `"use client"` components. Reads from the browser's storage.
- `middleware.ts` — for `src/middleware.ts`. Refreshes the session and sets cookies.

Using the wrong one causes auth to silently fail. Match the variant to the runtime.

### Cookies and `await`

In Next.js 15+, `cookies()` returns a Promise. Always `await` it:

    const cookieStore = await cookies();

If you see "cookies() should be awaited" warnings, you've missed an `await`.

### Server Actions must be marked `"use server"`

A Server Action file starts with `"use server";` (a directive, not an import). Forgetting this turns the file into a regular module that can run anywhere. The runtime won't catch this — you'll get cryptic errors.

Files in `src/lib/actions/` are always Server Action files; the directive goes at the top.

### `revalidatePath` after every mutation

After a mutation, call `revalidatePath('/path-being-updated')` so that the next render fetches fresh data. Otherwise the UI shows stale data after the user navigates back.

For the route list, revalidate `/routes`. For a specific route detail, revalidate `/routes/[id]`. Path matters — over-revalidation hurts performance, under-revalidation produces stale UI.

### Form submission with progressive enhancement

Forms should work even with JavaScript disabled (progressive enhancement). Use `<form action={serverAction}>` not `onSubmit` handlers. If you need client-side logic (e.g., showing a loading state), wrap with `useTransition` or use a small client component for that interactivity.

### Redirects in Server Actions

To redirect from a Server Action, throw `redirect('/path')` from `next/navigation`. Do not `return redirect(...)` — `redirect` works by throwing, and returning swallows the throw. The TypeScript type signature looks like it returns `never`, which is correct.

### Supabase RLS errors are silent

If RLS rejects a query, Supabase returns an empty result set, not an error. This can be confusing — you query the database, get nothing, and your code happily renders an empty list. Debug by running the same query in the Supabase SQL editor (which bypasses RLS as a superuser).

If an authenticated user "can't see" data they should see, check the RLS policy first.

### Custom access token hook is a manual Supabase Auth console step

The hook that stamps `operator_id` into the JWT custom claims is **not** a database migration — it is a manual action in the Supabase Auth dashboard (see Setup Notes). After a project reset, environment switch, or fresh setup, RLS-scoped queries will return empty until the hook is re-registered. **Check this first** before debugging RLS policies or client code when "nothing shows up."

### `stops_content_hash` is server-computed

The hash is computed inside the `replace_route_with_stops` RPC, never by the dashboard. Don't compute it client-side and pass it in — the hash is the integrity contract for FR-WD-12 divergence detection (Rule 12). Trust the RPC.

### Anonymous Supabase Auth users accumulate

Each successful tablet pairing creates an anonymous Supabase Auth user. There is **no automatic cleanup** in the initial release — deregistering or replacing a tablet leaves the auth user behind. Deferred operational concern; flagged for future hardening. Don't panic when `auth.users` grows beyond the dashboard-user count.

### Generated types vs hand-written types

Supabase types are in `src/lib/types/database.ts`. They are regenerated from the schema; do not hand-edit them. If you need a derived type (e.g., "RouteWithStops"), define it in a separate file:

    // src/lib/types/route.ts
    import type { Database } from './database';
    type Route = Database['public']['Tables']['routes']['Row'];
    type RouteStop = Database['public']['Tables']['route_stops']['Row'];
    export type RouteWithStops = Route & { stops: RouteStop[] };

### `next/image` quirks with Supabase Storage

If you ever load images from Supabase Storage (e.g., operator logos in a future release), `next/image` requires the Supabase domain to be allowlisted in `next.config.js`:

    images: {
      remotePatterns: [{ protocol: 'https', hostname: '*.supabase.co' }]
    }

Not relevant for the initial release (no images) but worth knowing.

### Vercel deploys preview branches automatically

Every push to a branch deploys a preview URL. Do not include secrets in PR descriptions or commit messages — they'd be visible in the Vercel build logs.

Production environment variables are set in Vercel's dashboard, not in the repo. The user manages this.

### Render-then-FCM ordering must not be bypassed

If a future task tempts you to "just fire FCM directly from the dashboard so the tablets refresh faster" — don't. The tablets will 404 trying to download audio that hasn't been rendered yet. The `audio-render-worker` is the only component that dispatches the post-save FCM push, and only after render success. See Rule 10.

### Audio pipeline is LINEAR16/WAV end-to-end — no MP3 anywhere

Round-3 Task 2 switched the audio pipeline from MP3 to LINEAR16 PCM, 24 kHz mono, WAV end-to-end. The Google Cloud TTS API call uses `audioEncoding: 'LINEAR16'`; the worker writes `.wav` files to the `route-audio` Storage bucket; the tablet plays `.wav` files. Why: Reg 13(4) requires preservation of a specific frequency range (300–3000 Hz) and MP3 perceptual coding cannot be relied on to preserve it without per-file verification. LINEAR16 is lossless and verification is empirical (see `spike-records/round-3/findings-tts-frequency.md`). File size is the cost: WAV files are ~10× larger than MP3 (~240 KB per 5-second stop announcement); storage and bandwidth budgets account for this. If you see `.mp3` in a path, a Storage call, or an Edge Function argument anywhere in the pipeline, that is a bug.

### FR-WD-21 Re-Render Audio has two-layer deduplication

The dashboard's Re-Render Audio button (FR-WD-21) has two-layer deduplication and you must implement both:

- **Client-side (UX only).** When the button is clicked, disable it and show a spinner for **60 seconds**, OR until the route's `audio_render_status` flips to `ok` or `failed` — whichever comes first. This prevents the user from spamming the button while a render is in flight and produces the impression of immediate response.
- **Server-side (authoritative).** The `enqueue-render-job` Edge Function checks pg_boss for an existing `created` or `active` job at the same `(route_id, route_version)`. If one exists, it short-circuits and returns the existing job ID without enqueuing a duplicate.

The server-side check is the actual deduplication. **Do not skip it** because "the client-side disable handles it" — a user with two tabs open, an over-keen retry, or a future automated trigger could bypass the client-side disable and the server would then dispatch redundant FCM pushes. The client-side disable is purely UX.

### `pg_boss` is configured `{ supervise: false, schedule: false }` — never change this

Rule 10 names this as inviolable. If you find yourself reading pg_boss documentation that says "you must call `boss.start()`" without these options, that documentation is for long-lived host processes. The Deno Edge Function isolate is short-lived and scheduled externally, and the supervisor/scheduler loops conflict with that lifecycle. Maintenance work that the disabled supervisor would have done (archival, expiry, retention) is handled by the `pgboss-maintain` scheduled Edge Function running hourly. Verified in `spike-records/round-3/findings-pg_boss.md`.

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

## Coordination with the Android App

The dashboard and the Android app are in separate repositories but share:

- **The Supabase project and schema**, including RLS policies and the `custom_access_token_hook`.
- **Tables:** `operators`, `devices`, `routes`, `route_stops`, `naptan_stations`, `device_pairing_codes`, `journey_summaries`, `rate_limit_attempts`.
- **Edge Functions** called from the dashboard: `generate-pairing-code`, `pair-device`, `recover-device`, `enqueue-render-job`. The dashboard does NOT call `render-route-audio` — that name is now the `pg_boss` job name, not an Edge Function; the dashboard interacts with it only by enqueuing via `enqueue-render-job`.
- **Scheduled Edge Functions** (not invoked from the dashboard but part of the shared backend): `audio-render-worker` (polls the `pg_boss` queue, consumes `render-route-audio` jobs, calls Google Cloud TTS, writes versioned WAV files to `route-audio`, dispatches the post-render FCM push; constructed with `{ supervise: false, schedule: false }` — see Rule 10) and `pgboss-maintain` (hourly `boss.maintain()` call).
- **Storage bucket:** `route-audio` (object scheme: `{operator_id}/{route_id}/{route_version}/...`, file format: LINEAR16 PCM 24 kHz mono WAV). Written by `audio-render-worker`; read by tablets. Retains the latest three versions per route.
- **Auth model:** Supabase Auth users for both operators (regular email/password) and devices (anonymous, created by `pair-device`). The custom access token hook stamps `operator_id` into the JWT custom claims for both — RLS depends on this.

If a feature in the dashboard implies a schema change, that change affects the Android app too. The architect coordinates schema changes deliberately — never modify the database schema as a side effect of a dashboard task.

---

## What This File Is Not

This file is **not** a specification. It does not describe what the dashboard does. The PRD describes that. This file describes **how to work on the dashboard**.

If you need to know "what does feature X do?", ask the user. If you need to know "how do I implement features in this codebase?", read this file.

---

## Sweep Summary (v3.9 changelog detail)

Round-3 deltas (v3.8 → v3.9) now reflected:
- **Audio file format switched MP3 → LINEAR16 WAV** end-to-end (Rule 10 narrative, the new "Audio pipeline is LINEAR16/WAV end-to-end" gotcha, the coordination list Storage bucket entry). Google Cloud TTS call uses `audioEncoding: 'LINEAR16'`; bucket holds `.wav` files.
- **pg_boss configuration inviolable rule** (`{ supervise: false, schedule: false }`) named in Rule 10 and in a new dedicated gotcha. Verified in `spike-records/round-3/findings-pg_boss.md`.
- **`audio-render-worker` reframed as a scheduled Supabase Edge Function** (corrects v3.8 Setup Notes that said it was a separately-deployed runtime). Confirmed empirically by the round-3 spike.
- **`pgboss-maintain`** scheduled Edge Function added to the coordination list and Setup Notes (hourly `boss.maintain()` call).
- **Three-version Storage retention** (was two) named in Rule 10 and the coordination list.
- **FR-WD-21 two-layer deduplication** gotcha added (client-side 60s disable + server-side job check inside `enqueue-render-job`).
- **FR-WD-19 cross-reference corrected** in Rule 11 — the audio-toggle reference now points at FR-WD-22 and §5.2 (FR-WD-19 was renumbered to HTTPS in round 2).
- **Operator-practice note** added to Rule 11: PRD §11.2 item 9 names the verify-`audio_render_status`-before-service practice; FR-WD-13 global failed-render indicator is the surface.
- **Staging environment setup** added to Setup Notes: two Supabase projects, two Vercel targets, six Sentry DSNs, with ordering dependencies named (and Data-Arch §12 cross-referenced).
- **Reg 13(4) frequency-range empirically verified** (no longer a pre-deployment blocker) — named in Rule 10.

---

## Sweep Summary (v3.8 changelog detail)

Round-1 deltas (v3.0 → v3.7) now reflected:
- Operator-status three-state enum (`pending` / `active` / `suspended`) replaces boolean `is_approved`; `/suspended` route added (Rule 4).
- Per-device `audio_enabled` boolean with dashboard fleet-view toggle (Rule 11).
- `routes.audio_render_status` / `audio_version` / `audio_announcement_hash` columns and the journey-start-gate semantics on the tablet side (Rule 10).
- `route-audio` Storage bucket added to coordination list.
- FR-WD-12 return-route divergence detection introduced (then re-engineered in round 2 — see below).
- Per-stop `proximity_radius_meters` and per-stop `segment_type` documented in Rule 6 surface.

Round-2 deltas (v3.7 → v3.8) now reflected:
- Audio pipeline rebuilt: dashboard calls `enqueue-render-job` (Edge Function), `audio-render-worker` consumes the `render-route-audio` `pg_boss` job, Google Cloud TTS (voice `en-GB-Neural2-B` locked), version-keyed Storage paths, render-then-FCM ordering (Rule 10). The deprecated synchronous `render-route-audio` Edge Function is removed.
- `routes.stops_content_hash` computed inside `replace_route_with_stops`; drives structural FR-WD-12 divergence detection (Rule 12). The old `last_synced_with_return` timestamp mechanism is deprecated.
- `audio_enabled` framing made honest — per-device toggle + non-blocking warning, no system enforcement of one-per-bus (Rule 11).
- Operator suspension is honoured at journey end on the tablet (Rule 4 narrative).
- Sentry crash telemetry adopted on the dashboard (server, client, and edge runtimes).
- `journey_summaries` table added to shared backend list.
- `audio_enabled = false` tablets skip audio downloads entirely (clarified Rule 11).
- `devices.status` renamed `devices.activation_state` (CHECK constraint).
- `rate_limit_attempts` table added to shared backend list.
- Hard-delete language removed (no scheduled hard-deletes in initial release; `ON DELETE SET NULL` retained as defence).
- Custom access token hook documented as a manual Supabase Auth console setup step (Setup Notes + Known Gotchas).
- Anonymous Auth user accumulation acknowledged as deferred operational concern.

---

## End of CLAUDE.md
