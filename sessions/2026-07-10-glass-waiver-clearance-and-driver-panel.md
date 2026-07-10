# Session: 2026-07-10 — Item-5 waiver set cleared on glass + driver panel expansion (fifth session)

**What changed:** The entire item-5 residual-risk set was glass-verified before the pilot, in nine small owner-driven runs on the Lenovo tablet against staging, each cross-checked server-side via read-only MCP. Setup first (owner, dashboard): hail-and-ride restored on "Marford to Guildhall" stops 2–3 (Marford Shops, Angel Inn) and a return route generated (`6820d7bb`, reversed stops, H&R mirrored, cross-linked `return_route_id`s, rendered ok both). The same straight-through Marford→Guildhall Lockito route served every run. Then, owner-requested, the **driver control panel was expanded and regrouped** (two commits): wrap-content height (was a fixed ~182dp cap forcing a scroll), width-capped centred control pairs, End Journey isolated at the bottom — deliberately superseding the panel's original "never covers the hero" cap under the existing driver-overlay occlusion posture (new ledger #24).

**The nine runs (all pass):**
1. **Plain end-to-end H&R re-baseline** on the edited route — route/stop announcements, section start/end at the true boundaries, silence within the section, tube-map diamonds+dashed, termination; summary 6/6, zero anomalies.
2. **H&R-silent landing** (skip Marford Library → lands on H&R Marford Shops): exactly one section-start, no next-stop audio at all, `STOP_SKIPPED` row; `diversion_invoked = true`.
3. **Skip across the section-end boundary** (skip Fairfield Street from H&R Angel Inn): one net `SECTION_ENDED` naming the *landing* (Guildhall Square) — boundary keyed on true from→landing, neither swallowed nor doubled; journey end with diversion still active is clean.
4. **Manual stepping with a skip set active:** manual Next steps *onto* the struck-through stop and plays its next-stop audio (manual is authoritative; no `STOP_SKIPPED` from manual), Previous is a silent rewind, GPS afterwards re-applies the skip; summary `manual_advances_count = 1`.
5. **Zero-stop diversion commit:** announcement fires, no strikethrough, panel never enters "active", no rows — and the summary latch stays **false** (latch tracks a non-empty skip set only).
6. **Manual H&R buttons:** override suppresses next-stop on *scheduled* stops AND both GPS boundary announcements (a run of total silence through all four stops); END lifts it; non-transition re-presses of **both** buttons announce in full (always-fire — the START re-press was an owner accident that improved coverage); four `DRIVER_HAIL_RIDE` rows, zero GPS section rows.
7. **Manual END inside a physical H&R section:** always-fires, then the real Angel Inn boundary fires the documented-expected second "leaving" announcement (one DRIVER_HAIL_RIDE row + one GPS row).
8. **Skip-to-terminus** (skip Fairfield + Guildhall): journey terminates at Angel Inn — termination fires, no section-end (null approached ⇒ termination owns the end of the line), two `STOP_SKIPPED` rows.
9. **Return-journey `active_route_id` flip**, verified by UUID at all three states: outbound set → return route set (via terminus completion → end-screen "start return journey") → null after end. Bonus: an early driver-panel End Journey also cleared it, and even a 20-second false-start journey uploaded a clean summary.

**Commits:**
- pds-android `e34414f` — feat: expand the open driver panel so the full control stack shows without scrolling (FR-AT-40)
- pds-android `67f8412` — feat: driver panel wraps its control stack exactly and regroups the controls (FR-AT-40)
- pds-planning (this commit) — session log + STATE.md close-out + working-file removal

(Both code commits reviewed and pushed by owner.)

**Decisions made:**
- **Driver-panel occlusion posture (ledger #24):** the open panel may now cover part of the hero while open. Justified by the precedent already accepted for the full-size `JumpToStopOverlay` / `StopSelectorOverlay` driver surfaces; the Reg 7/9 guards are unchanged (composes below the regulated flash/overlay, collapse-on-fire, driver-opened only, 30 s auto-dismiss). **Glass-verified during this session:** an H&R start announcement took the screen over the open panel exactly as designed.
- **Waiver item "drop-overlap on-button confirmation" recorded as not reproducible by construction:** the panel collapses whenever a regulated announcement fires, so a driver physically cannot stack a second press; the guarded race (GPS-fired regulated announcement colliding with a press) can't be staged by hand. The load-bearing property — state-flip → event-row → announce ordering in `TriggerManualHailRideUseCase` — is unit-covered. Not a pass, not an open action.
- End journey behaviour confirmed as-designed, not a bug: driver-panel End Journey (FR-AT-43) routes to the route list; only terminus completion (FR-AT-16) shows the journey-end screen with the return-journey offer.

**Verified:** All nine runs on the physical tablet (Lockito GPS, accuracy under the 80 m radii), each with the admin event-log screenshot reviewed against per-run row predictions and staging cross-checks (`journey_summaries` counts / `diversion_invoked` / `uploaded_at` sub-second drains; `devices.active_route_id` at every state). The two panel commits glass-checked by the owner, including collapse-on-fire over the open panel and the diversion-active three-button row. Builds + full unit suite green after each commit.

**What's next:** The pilot. No build items, no banked pre-pilot work, and — as of this session — **no glass debt**.

**Banked / open:**
- **New docs nit:** the Android CLAUDE.md says `journey_events` are "cleared at both app startup and journey start"; the implementation is a **30-day retention prune** (`CleanupJourneyEventsUseCase`, FR-AT-63), so past-journey rows legitimately persist in the admin event-log viewer. One-line docs fix for a future docs commit.
- **Process note (owner-flagged):** the working glass-test script was initially written into `living/` (`glass-verification-script.md`) — the wrong home for a transient working document; `living/` is for STATE.md, the durable snapshot. Deleted at this close-out (it was never committed). Transient working material belongs in scratch space or the conversation, not the planning repo.
- Everything else unchanged (accepted postures only).
