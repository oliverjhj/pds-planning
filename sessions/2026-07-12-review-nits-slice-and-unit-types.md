# Session: 2026-07-12 — Review-nits slice + Mm/Px/Ppmm unit types

**What changed:** A cleanup session (sixth same-day 2026-07-12 session) that cleared the entire "small nits banked (fix opportunistically)" list from STATE.md, then executed the previously-deferred e7. Two plan-mode-gated slices, both owner-reviewed/pushed and owner-glass-verified.

**Slice 1 — the banked nits (plan approved, full sweep of review findings a–e):**
- **Doc-nit (dashboard, `43a6852`):** corrected "8 Edge Functions" → **9** (`delete-account`) across `CLAUDE.md` (tree comment, Coordination list + a `delete-account` entry, Sentry-count, deployment-state line — the 2026-07-09 "8 deployed" fact kept accurate, the 9th noted as added 2026-07-12) and the `_shared/sentry.ts` header comment.
- **CalibrationScreen scroll (`cf26973`):** added `verticalScroll` to `SliderStep` + `VerifyStep` (parity with `PreviewStep`) so the confirm button stays reachable on short-aspect displays.
- **Finding (a) — overlay re-key (`48bbe92`):** `RegulatedAnnouncementOverlay`'s scroll/hold `LaunchedEffect` was keyed only on `lastText`; re-keyed on the same tuple as the `plan` `remember` (`lastText, availableWidthPx, viewportHeightPx, resolvedPpmm`) so a mid-overlay viewport/ppmm change re-reports the hold and restarts the scroll. Compliance-adjacent.
- **Finding (c) — first-paint gate (`0a9a783`):** `calibrationPpmm` is null both before the async read AND when genuinely uncalibrated, so a calibrated tablet flashed a DisplayMetrics-fallback size for the first frame(s). Added an explicit `calibrationLoaded: StateFlow<Boolean>` in `JourneyViewModel`; `JourneyScreen` holds first paint behind a bare ground box until true (same stance as `CalibrationStep.Loading`). Value semantics unchanged.
- **Finding (b) — line-height dedup (`d04d01e`):** the `1.25f` multiple lived as two comment-tied literals; moved the single source to `PhysicalTextMath.LINE_HEIGHT_MULTIPLE`, both existing names now alias it.
- **Finding (d) — tests (`f7221ad`):** extracted the Tier-C Reg 10(1) carrier logic (diversion gate + order-by-index + de-null) into pure `PassengerTierMath.tierCSkippedStopNames` (behaviour-preserving) and tested it; literal-pinned the ticker/overlay speed/gap/hold constants + the just-over ticker overflow boundary.
- **Findings (e1+e2) composition hygiene (`7235674`):** admin PIN fields `rememberSaveable`→`remember` (PIN never serialized into the saved-instance Bundle); the overlay's `lastText = text` composition-time write moved into a `SideEffect`.
- **Findings (e3+e4) memoisation (`fdb0f89`):** wrapped `HeroNextStop`'s per-recomposition measure ladder in `remember`; wrapped `rememberPhysicalTextStyle`/`Size` results in `remember` (they were named `remember*` but rebuilt every recomposition).
- **Finding (e6) modifier params (`c21d12c`):** added `modifier: Modifier = Modifier` to the four private layout-emitting journey helpers (`RegulatedFlashOverlay`, `RegulatedAnnouncementOverlay`, `PassengerInfo`, `CombinedJourneyTicker`).
- **(e5) reviewed, no change** (the four `DisposableEffect(Unit)` sites are correct view-lifetime effects). **(e7) deferred** to slice 2.

**Slice 2 — e7 unit value classes (`7534746`, one atomic commit):** introduced `Mm` / `Px` / `Ppmm` `@JvmInline value class`es (new `util/Units.kt`) and applied them across the pure text-math util — `PhysicalTextMath` (`fontSizePxForPhysicalHeight`, `overlayScrollPlan`, `tickerPlan`, the ppmm derivations, the `*Px` plan fields) and `PassengerTierMath` (`lineHeightPx`, all ten `TierInputs` px fields, `PassengerLayoutPlan`). `ppmm` is a ratio so it got its own `Ppmm` type; rates (mm/s, px/s), durations (ms), dpi and dimensionless fractions stayed primitive. The ~6 presentation call sites wrap/unwrap at the util boundary; the nullable composable `ppmm: Float?` / `targetMm: Float` params stayed `Float` so nothing boxes in the frame-rate path (per the `kotlin-types-value-class` skill). Delegated `ppmm` reads use `?.let` (a delegated property can't be smart-cast). ~45 test call sites updated (tests live in the same package, no imports).

**Commits:**
- pds-dashboard `43a6852` — docs: Edge Function count 8 → 9 (delete-account)
- pds-android `cf26973` — fix: scroll CalibrationScreen slider & verify steps (FR-AT-35)
- pds-android `48bbe92` — fix: re-key regulated-overlay scroll/hold effect on layout inputs (finding a)
- pds-android `0a9a783` — fix: gate passenger first-paint until calibration loads (finding c, Rule 13)
- pds-android `d04d01e` — refactor: single source for the 22 mm line-height multiple (finding b)
- pds-android `f7221ad` — test: Tier-C Reg 10(1) carrier + pin ticker/overlay constants (finding d)
- pds-android `7235674` — refactor: stop writing PIN & overlay text as composition-time state (e1+e2)
- pds-android `fdb0f89` — perf: memoise hero measure ladder & physical text style (e3+e4)
- pds-android `c21d12c` — refactor: modifier params on journey overlay composables (e6)
- pds-android `7534746` — refactor: introduce Mm/Px/Ppmm unit types in the text-math util (e7)

**Decisions made:**
- **Accepted postures stay in STATE.md** (owner-confirmed): the "do-not-fix" postures (journey_end drop-as-AlreadyRunning, FCM-secret log-only, dashboard beforeSend out of scope, delete-with-data unexercised) live in the living doc on purpose — their whole function is to stop a future session re-opening settled behaviour; burying them in a session file would erode that. Filed under "Bank and Assess / Accepted postures", not "What's Left".
- **e7 done as full util typing** (owner chose over core-only or skip): coherent coverage; `ppmm` needed a third type (`Ppmm`); behaviour-identical (value classes are zero-cost, so the unit tests are the proof).
- **First-paint gate** uses a separate `calibrationLoaded` boolean rather than overloading the nullable ppmm — distinguishes "not loaded" from "genuinely uncalibrated".
- **(e5) no-op, (e7) full, (e6) lowest-value-but-included** — scope explicitly agreed in plan review.

**Verified:** Dashboard docs — no build (grep-confirmed no stale "8"). Android — `compileStagingDebugKotlin` + `testStagingDebugUnitTest` + `assembleStagingDebug` green across the work; the e7 tests prove the maths byte-identical. **Owner glass-verified on the Lenovo** (`HA2DG5NK`) and confirmed pass: the three render-path nits (CalibrationScreen, overlay re-key, first-paint gate) and the e7 refactor. One false alarm mid-session — owner reported lag + a calibration-back regression, investigated (nav reopen is a durable StateFlow, not droppable; no static cause found in the diffs), then owner reconfirmed it was imagined and everything is fine.

**What's next:** The **payment gate** (What's Left #7 — the one substantive build item) then the **pilot**, owner-led. Unchanged.

**Banked / open:** Nothing from this session. The "small nits banked" list is **fully cleared** (e7 was its last spun-out item). Post-pilot optional backlog is unchanged (repeat-last-announcement, ANDROID-3 self-heal, fleet-view auto-refresh, server-alignment hardening set, staging SMTP, email-template rebrand, Speed Insights, GitHub Actions CI, @sentry/deno v10).
