# CLAUDE.md
# Passenger Display System (PDS) — Web Dashboard

This file is your reference for working on the PDS web dashboard. Read it at the start of every session. It contains conventions, architectural rules, and known gotchas. If a rule here conflicts with a prompt, ask before deviating.

---

## What This Project Is

A Next.js web dashboard where UK bus operators register their company, manage their fleet of Android tablets, and author the routes that those tablets display. It is the cloud-side companion to a separate Android application (different repository).

Bus operators sign up via this dashboard, get manually approved by the system administrator, then use the dashboard to create routes, generate device pairing codes, and monitor their fleet.

The product is legally regulated under the UK Public Service Vehicles (Accessible Information) Regulations 2023. The dashboard contributes to compliance indirectly: the routes authored here are what tablets display to passengers, so accuracy matters.

---

## Tech Stack

- **Framework:** Next.js 15+ with App Router (not Pages Router)
- **Language:** TypeScript (strict mode, no `any` without justification)
- **Hosting:** Vercel (free tier)
- **Backend:** Supabase (shared with the Android app — same project, same tables)
- **Auth:** Supabase Auth (email + password)
- **Data fetching:** Server Components and Server Actions; minimal client-side Supabase SDK use for interactive bits
- **Styling:** Tailwind CSS
- **Components:** shadcn/ui (Radix UI primitives + Tailwind)
- **Forms:** React Hook Form + Zod for validation
- **Icons:** Lucide React (bundled with shadcn/ui)

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
    │   │   ├── devices/              # Device fleet view, pairing
    │   │   ├── account/              # Account settings
    │   │   ├── pending/              # "Pending approval" landing
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
    │   │   ├── devices.ts            # generatePairingCode, deactivateDevice, renameDevice
    │   │   └── account.ts            # updateProfile, changePassword
    │   ├── schemas/                  # Zod schemas for validation
    │   ├── types/                    # Shared TypeScript types (often generated from Supabase)
    │   └── utils/                    # Generic utilities (formatDate, cn, etc.)
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

## Architectural Rules — DO NOT VIOLATE

These are the rules whose violation breaks the architecture. Treat them as inviolable.

### 1. Server Components by default, client components only when needed

Next.js App Router is Server Component-first. Every component you write defaults to running on the server. Only add `"use client"` when you genuinely need client-side interactivity (forms with complex state, drag-and-drop, charts, anything using browser APIs).

If you find yourself adding `"use client"` to the top of every file, stop and reconsider. Most of the dashboard is data display, which is a server-component case.

### 2. Mutations are Server Actions, not API routes

For any operation that changes data (create/update/delete route, generate pairing code, etc.), use a Server Action. Do not create a `/app/api/` route handler unless you have a specific reason (e.g., a third-party webhook).

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
      // ... insert into Supabase
      revalidatePath("/routes");
    }

### 3. Auth checks at the layout level, not per-component

Every protected route is under `(dashboard)/layout.tsx`. That layout fetches the user once via `getUser()` and either renders children or redirects. Individual pages do not re-check auth.

The middleware (`src/middleware.ts`) refreshes the Supabase session on every request to keep tokens fresh. The layout does the actual gating.

### 4. The `is_approved` gate is enforced server-side

After auth, every dashboard action must check that the operator's `is_approved = true`. This is enforced by RLS at the database level (queries return empty for unapproved operators), and additionally by the dashboard's routing logic:

- The `(dashboard)/layout.tsx` checks `is_approved` after auth.
- If false, the user is redirected to `/pending`.
- The `/pending` route shows the "account pending approval" message.
- Account profile pages remain accessible even when pending; everything else is gated.

Do not implement "soft" approval gates in individual components. The single chokepoint is the dashboard layout.

### 5. Supabase queries respect RLS — do not bypass it

The dashboard always uses the user-scoped Supabase client (with the user's session JWT). Never use the service role key from the client or from regular server actions. RLS policies are the authoritative security boundary.

The service role key is used only by Edge Functions (which run in Supabase's environment, not in Next.js). If a dashboard operation seems to require service role privileges, that operation belongs in an Edge Function, not in a Server Action.

Examples requiring Edge Functions, not Server Actions:
- Generating pairing codes (writes to `device_pairing_codes` are service-role-only)
- Creating anonymous Supabase users (admin API)
- Modifying device JWTs

The dashboard calls Edge Functions via `supabase.functions.invoke()`.

### 6. Routes are authored on the dashboard, not the tablet

The tablet is read-only for routes. The dashboard is the **only** authoring surface. All CRUD on routes happens here.

This means the dashboard's route builder is the most feature-rich UI in the product. It must handle: NaPTAN search, drag-and-drop stop reordering, return route generation, route metadata, validation, error states. Build this carefully.

### 7. NaPTAN data is queried, not bundled

The dashboard queries NaPTAN data from the Supabase `naptan_stations` table via Postgres full-text search. Do not download the entire NaPTAN dataset to the client. The table has ~400,000 rows; only return what matches the user's search query.

Search is debounced (250ms typical). Results limited to ~20 hits per query.

### 8. Stop data is copied into route_stops at creation

When the user adds a stop to a route, the NaPTAN ID, name, CRS code, latitude, and longitude are *copied* into the new `route_stops` row. The `route_stops` table does not foreign-key to `naptan_stations`. This means routes survive NaPTAN data changes (a station rename in NaPTAN does not affect saved routes).

This is the same rule as the Android side (rule 4 there). The behaviour must be consistent across both surfaces.

### 9. Use the atomic route + stops RPC for saves

When saving a route (create or update), call the `replace_route_with_stops` Supabase RPC. This is a single transactional operation that upserts the route and replaces all its stops atomically.

Do not:
- Insert the route and then insert stops in separate queries (race condition)
- Delete stops before inserting new ones in separate queries (deletes orphan stops if insert fails)

Always use the RPC.

### 10. All timestamps in UTC

Supabase stores `TIMESTAMPTZ` in UTC. When displaying timestamps to the user, convert to UK local time (handles GMT/BST) using `Intl.DateTimeFormat` with `timeZone: 'Europe/London'`. Never use raw `Date` formatting that depends on the server's locale.

### 11. Forms validated with Zod

Every form has a Zod schema in `src/lib/schemas/`. Server Actions parse `FormData` through that schema before touching Supabase. Client-side validation (via React Hook Form's Zod resolver) provides immediate feedback; server-side validation is the authoritative check.

Schemas are shared between client and server. Do not duplicate validation logic.

### 12. TypeScript strict mode

The project is in TypeScript strict mode. `any` is banned without a written justification comment. Supabase types are generated from the schema using `supabase gen types` — use the generated types, do not write your own.

When the generated types are stale (e.g., a migration added a column), regenerate them. Don't paper over the gap with `as any` casts.

---

## Key Data Decisions

**One Supabase project, two surfaces.** The dashboard and the Android app share the same Supabase project, the same tables, the same RLS policies. Schema changes affect both. Coordinate via the architect.

**Email/password auth for operators.** No magic links, no social login, no MFA in the initial release. Supabase Auth handles the flow (signup, email verification, password reset). Use the default Supabase Auth UI flows or build matching screens.

**Anonymous users for devices.** Tablets pair via Edge Functions that create anonymous Supabase Auth users. The dashboard never creates these users; it only triggers the flow by generating pairing codes. Devices are linked to operators via the `devices` table's `operator_id` and `user_id` columns.

**Operator → user mapping is one-to-one.** Each operator has exactly one Supabase Auth user. No team accounts. If multiple people at a bus company need access, they share the login.

**Soft deletes for routes.** Deleting a route sets `is_deleted = true`, not a hard `DELETE`. This is required for the tablets' sync algorithm — deleted routes propagate as tombstones. Always use the dashboard's delete action; never issue a hard DELETE.

**NaPTAN search via Postgres full-text.** The `naptan_stations.search_vector` column is a generated `TSVECTOR`. Queries use `search_vector @@ websearch_to_tsquery('english', :query)`. Results ranked by `ts_rank`. Return the top 20.

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

These are concrete problems that often bite Next.js + Supabase dashboards. Avoid them.

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

- The Supabase project and schema
- The `routes`, `route_stops`, `devices`, `operators`, `naptan_stations`, and `device_pairing_codes` tables
- The Edge Functions (`generate-pairing-code`, `pair-device`, `recover-device`, plus RPCs)
- The auth model (Supabase Auth users for both operators and devices)

If a feature in the dashboard implies a schema change, that change affects the Android app too. The architect coordinates schema changes deliberately — never modify the database schema as a side effect of a dashboard task.

---

## What This File Is Not

This file is **not** a specification. It does not describe what the dashboard does. The PRD describes that. This file describes **how to work on the dashboard**.

If you need to know "what does feature X do?", ask the user. If you need to know "how do I implement features in this codebase?", read this file.

---

## End of CLAUDE.md
