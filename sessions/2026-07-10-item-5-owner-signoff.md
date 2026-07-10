# Session: 2026-07-10 — Item 5 close-out: owner sign-off on partial glass verification

**What changed:** No code. Item 5 (the consolidated full-app GPS glass pass) was prepared, partially executed, and closed by owner decision. Claude produced the full three-run test script, grounded in the code (Explore pass: detection candidates stay raw `stops[index]`/`stops[index+1]` and the skip-walk applies to the *approached* index at the pre-skip stop's announcement, so one straight-through Lockito route serves every run) and in the live staging stop layout of route `575261eb` (H&R section = stops 2–3, all 80 m radii). The owner ran **one plain end-to-end Marford→Guildhall run — passed** — and did not execute the diversion or manual-H&R runs. The runs were initially reported as all passing; the owner then corrected the record in-session and directed that item 5 be signed off as complete with the true state recorded as a caveat.

**Commits:** pds-planning (this commit) — session log + STATE.md close-out.

**Decisions made:**
- **Item 5 closed by owner sign-off on partial verification (owner-directed).** Rationale, recorded per the owner's instruction: the build stage needs to end; the owner is moving to **iterative testing** (exercise features one at a time, go back and fix whatever surfaces) and trusts the unit-green, code-verified implementation. Residual risk accepted: the three skip-set diversion cases (plain mid-route skip, H&R-silent landing, manual-override), the zero-stop diversion commit, and all manual-H&R button behaviours (next-stop/boundary suppression, always-fire non-transition END, drop-overlap on-button confirmation, `DRIVER_HAIL_RIDE` event rows) remain **unit-green + code-verified only, not glass-verified**. Deliberately recorded as done-with-caveat, **not** an open action — if something is broken, iterative testing is expected to surface it.
- **One-off waiver, not precedent.** This does not weaken the hardware-verification discipline (workspace rule / ledger #7), which remains in force for future compliance-touching changes.
- Claude's stated counterweight, for the record: diversions and manual H&R are driver-invoked exception paths that casual runs won't exercise, and they are regulated announcements — worth including in the owner's focus-testing before the pilot. Recorded as observation only, per the owner's instruction.

**Verified:** One plain GPS run on the Lenovo tablet (route `575261eb`, Lockito) — passed end-to-end: route announcement, stop announcements, H&R next-stop silence, GPS section start/end, termination; the tube-map H&R rendering (diamonds + dashed run) was on display throughout. This also covers the empty-set diversion case's GPS half by construction (normal advances behave normally on the shipped build). Everything else in the item-5 list: **not glass-verified** (see Decisions). The unexecuted three-run script is preserved in this session's conversation and is re-derivable from `StopDetectionInterpreter` / `TriggerManualHailRideUseCase` / `StartDiversionUseCase`.

**What's next:** Item 6 — Firebase staging/production project split, then the pilot. Working mode shifts from build-stage items to owner-led iterative testing.

**Banked / open:** (carried forward, plus one new)
- **New docs nit:** Android `CLAUDE.md` suggests `adb logcat -s PDS`, but all four production tags are dotted (`PDS.Sync`, `PDS.Fcm`, `PDS.Journey`, `PDS.Sentry`) — that filter matches nothing. One-line fix for a future docs commit.
- Journey-summary cross-check for the 2026-07-10 run not completed (upload pends the next sync; latest staging rows were 2026-07-05) — immaterial to the sign-off.
- ANDROID-3/4 Sentry issues (real July 1 tablet errors) — still worth a look before the pilot.
- Pre-existing lint error `scripts/render-fixed-announcements.ts:172`; render-worker unchecked returned-`error` gap; three stale dashboard text sites; deployed `pair-device` one-comment divergence — all carried unchanged.
