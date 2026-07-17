# Session: 2026-07-17 — Shipping-roadmap consolidation (third session)

**What changed:** The owner-requested comprehensive roadmap review (the ⚠️ first item flagged by the second session). `living/STATE.md` was substantially rewritten: **What's Left To Do is now "The Shipping Roadmap"** — six sequenced phases replacing the retired numbered build items (1–6 complete; item 7 → Phase 5; the item-8 raw intake dissolved into the phases). The owner's remaining to-do notes were intaken (driver-panel volume slider, diversion button layout, app-wide button sizing pass, tube-map transition visual) and the external to-do list is now cleared — STATE.md is the single execution plan. The **Current State section was condensed** from six session-narrative paragraphs (including the commit-stack paragraph) to five tight summaries; all removed detail remains permanently in the dated `sessions/` files, and the load-bearing operational facts live on in Operational Anchors / Bank and Assess / the Decision Ledger (all untouched).

**Commits:** pds-planning (this close-out commit) — docs: shipping-roadmap consolidation, STATE.md rewrite + session file.

**Decisions made (all owner-decided this session):**
- **Full design redo is happening and is the FIRST action (Phase 1):** replace the Booking Hall identity on BOTH surfaces with a black-and-white, clean, modern, simple, minimal design — one consistent identity across dashboard and tablet. A decided direction, not an exploration (the owner does not like the current colour scheme/design). Booking Hall rules remain in force until it lands; the redesign carries the Rules 16/20 rewrite, `PassengerPaletteTest` update, ledger supersessions (#25/#26 visual halves, #30), and the three-way design-system sync (ledger #27) in one coordinated move. Open sub-questions deferred into the design work: two-theme toggle survival, the semantic-warning colour in a B&W palette (ledger #19 constraint stands), Gelasio's fate.
- **Sequencing (owner-set):** design → polish/fixes → design-dependent authoring aids → public face/documentation → payment (deliberately late) → pilot readiness/pilot/launch gate. Pilot is calendar-driven ("asap" but no exact timeline — no-rush posture) and slides earlier when the bus company's calendar firms up; it does not depend on the payment or website phases.
- **Public landing page lives inside the existing Next.js app** as a pre-login front page (a separate site adds overhead for no current benefit; revisit only if marketing/SEO needs diverge).
- **Legal drafts (privacy documentation, signup T&Cs) are Claude's to draft**, owner reviews; the legal-entity question stays the owner's.
- **Repeat-last-announcement button reconfirmed deferred** post-pilot (not necessary; build later only if wanted).
- **Email deliverability package promoted** from Defer Without Guilt into Phase 6 (do before the pilot operator signs up); audit detail retained in the Defer entry.

**Verified:** Documentation-only session — no code, no builds, no hardware verification applicable. STATE.md reviewed section-by-section after the rewrite (seam blank-lines fixed; Decision Ledger, Bank and Assess, Defer Without Guilt, Operational Anchors confirmed untouched except the two noted Defer edits).

**What's next:** **Phase 1 — the design redo**, starting in a fresh chat: mockup/exploration rounds (the Booking Hall artifact-round pattern) so the owner reacts to concrete B&W directions before any code is touched.

**Banked / open:**
- Phase 1 open sub-questions (toggle, warning colour, typeface) — settle during design exploration.
- Repo CLAUDE.md files still cite the retired "item N" numbering — valid as historical reference (tombstone note in STATE.md); no renumbering.
- Dashboard CLAUDE.md mention of the geofence-warnings feature still owed on its next natural docs touch (carried from the second session).
