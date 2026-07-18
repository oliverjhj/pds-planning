# Session: 2026-07-18 — Route designation split + operator guide (Phase 3, part 1)

**What changed:** Split two fields whose roles had blurred, and gave operators authoring-time sight of the result. `routes.display_name` is now the operator's **internal** handle and never reaches a passenger; `routes.route_number` is the passenger-facing **designation** carrying Reg 7(1)(a) — free text, mandatory. Both surfaces now compose one template: screen `"{designation} service to {final stop}"`, audio `"This is the {designation} service to {final stop}."`, with the final stop always derived from the last stop. This killed a real content bug: the worker composed `"This is the {number} to {display_name}"`, so an A-to-B route name spoke as **"This is the X12 to Marford to Guildhall"** — a doubled "to", on every route named the natural way. The tablet's route line had the mirror problem (`"Marford to Guildhall — to Guildhall Square"`). Shipped alongside the wording fix, per the owner's call to combine them: a live preview in the route builder and a new `/guide` page. Deployed end-to-end and glass-verified on prod.

**Commits:**
- pds-dashboard `8646daa` — canonical passenger-announcement composition module (`_shared/announcements.ts` + golden tests + `@shared/*` aliases)
- pds-dashboard `6021144` — render worker announces "{designation} service to {final stop}"
- pds-dashboard `7150cdd` — (FR-WD-06) route designation mandatory and passenger-facing
- pds-dashboard `90e0cb2` — migration 026, `routes.route_number` NOT NULL
- pds-dashboard `05a5924` — live passenger-line preview in the route builder
- pds-dashboard `7199d6c` — operator guide page (+ middleware allowlist + nav, one commit)
- pds-dashboard `121982d` — CLAUDE.md: designation split, shared module, migration 026
- pds-android `39534db` — (FR-AT-12) passenger route line reads "{designation} service to {final stop}" (also drops the banner route number)
- pds-android `71f4ca0` — (FR-AT-42) route announcement preview mirrors the render worker
- pds-android `41980e1` — CLAUDE.md: route line composition and the designator fallback
- pds-planning `958528b` — design language: journey banner band carries no route identifier

**Decisions made:**
- **"service to", one template, both surfaces.** Chosen over plain "to" because the designation is free text: "the X12 service to X" and "the Marford Green service to X" both read correctly, whereas "the Marford Green to X" does not. It also already matched the frozen spec and the existing Tier C ticker string. `composeRouteAnnouncement` is literally `"This is the " + composeRouteLine(...) + "."`, so screen and audio cannot diverge.
- **One canonical module, in `_shared/`, imported by BOTH the Deno worker and Next.** Rejected keeping it in `src/` (reaching out of `supabase/` is outside the CLI's supported bundling surface) and rejected two copies pinned by a test (still permits a deploy between edit and test). Verified rather than assumed: `tsc --listFiles` shows `_shared/announcements.ts` **is** in the Next program despite tsconfig excluding `supabase/functions/**` — `exclude` only filters what `include` discovers — so the module is typechecked, which is what keeps Deno-isms out of it. Confirmed at deploy time too: the CLI uploaded `announcements.ts` alongside the worker on both projects.
- **Designation cap 20 → 40 chars** (owner). The old cap suited a route *number*; Reg 7(1)(a) accepts any label, and "Rail Replacement Service A" (26) would not fit.
- **Android keeps `route_number` nullable at every layer.** Making it non-null would be a Room schema bump (v3 + migration + the on-hardware migration ritual) for a column already non-null in practice. Also dissolves any cross-repo migration-ordering dependency. The null/blank fallback is centralised in the new pure `util/RouteLineMath.designator`.
- **The designator fallback is the repo's first NON-omitting passenger fallback.** Everywhere else the affordance simply disappears when absent (the driver-screen route-number chips); here omission would be non-compliant (Reg 7(1)(a) is required) and a blank would render `" service to X"`, so it falls back to `display_name`. Documented in both repos.
- **Guide is its own page at `/guide`, plus a live builder preview — both, not either.** They answer different questions: the preview answers "what will THIS route say?" while typing; the Guide answers "what does the system say at all, and when?" Rejected nesting under `/routes` (it documents fleet-wide announcements, and burying it there mislabels it), and rejected screenshots in favour of an HTML/CSS tablet mock that stays current and can be driven live by form values.
- **The tablet mock carries its own dark palette in both dashboard themes** — same posture as the product mark. It depicts a device, not a themed surface; a mock that turned white in light mode would misrepresent the hardware. New `--tablet-*` tokens mirror the Android `PassengerDark` palette rather than inventing values.
- **Banner route number removed** — the designation now leads the route line two rows below, so the band was showing the same value twice.

**Verified:**
- Dashboard: 44 vitest tests (19 new golden tests on the composition module), typecheck, lint, build — all green. `tsc --listFiles` used to *prove* the shared-module typecheck claim rather than assume it.
- Android: 6 new `RouteLineMathTest` cases confirmed to have actually run on both flavours (checked the JUnit XML, not just the green build); `./gradlew test assembleStagingDebug` green.
- **Security gate:** logged out, `/guide` redirects to `/login` — the middleware allowlist holds. (Without the `DASHBOARD_PATHS` entry the page would have been served anonymously, silently.)
- Migration 026 applied staging → prod, both pre-flights re-run immediately before apply (0 null, 0 blank on both). Post-apply verified via `pg_constraint`: `NOT NULL` + `routes_route_number_not_blank` present on both.
- Worker deployed to both projects; upload log confirms `_shared/announcements.ts` bundled.
- Prod route re-saved: `audio_version` 1784288894778 → **1784392762662** (increased), hash `236450ec28f8` → `14b6550a7b15` (changed), status ok, **7 WAVs confirmed at the new Storage path** (1 route + 6 stops).
- **On glass (owner, real Lenovo, prod build): audio correct and glass checks done.**

**What's next:** Phase 3's remaining item — the **ticker/marquee simulator** (deliberately out of scope this session). Then Phase 4.

**Banked / open:**
- **Staging's 4 live routes still carry the OLD audio** — they need a re-save each. Nothing breaks; staging just speaks the old wording until then.
- **No force-re-render for a healthy route.** `triggerReRender` hard-gates on `audio_render_status === 'failed'`, so re-saving through the builder is the only path for an `ok` route. That is why the one manual step tonight could not be automated. Fine at 1–5 routes; a real gap at fleet scale, and the concrete argument for building it.
- **Reported, not fixed (workspace rule):** dashboard CLAUDE.md Rule 10 claimed differential per-stop re-render "keyed on the stop name". The deployed worker has no per-stop hash and no copy path — every `stop_{N}.wav` is re-synthesised on every job. Only the **doc** was corrected; the code was left alone.
- **Cross-repo wording drift is a live residual risk** — nothing mechanically binds the Kotlin `strings.xml` wording to `_shared/announcements.ts`, and the failure mode is a tablet that shows one thing and says another. Mitigated only by bidirectional comments, the two mirrored test tables, and glass verification.
- The Supabase CLI link defaults to whatever was last linked; it sat on **production** at session start. See ledger #39.
