# Passenger Display System — planning, decisions, and build record

A legally-regulated passenger information system for UK rail-replacement buses: an Android tablet
application and a web dashboard over a shared Postgres backend, built to comply with **The Public
Service Vehicles (Accessible Information) Regulations 2023** (SI 2023/715).

**Planned March–May 2026 · built May–July 2026 · deployed and operated in production · commercial
track closed July 2026 · infrastructure since decommissioned.**

This repository is the project's documentation and decision record. The two application repositories
are private; this is the exhibit.

---

## Start here: the compliance matrix

**[`historical/Compliance-Mapping-Matrix.md`](historical/Compliance-Mapping-Matrix.md)**

Before any production code was written, the regulations were mapped clause by clause onto product
requirements. **36 individual clauses across eight regulations** (Regs 7–14), at sub-clause
granularity — `7(1)(b)(i)`, `10(2)(a)(ii)`, `14(5)(b)` — each in a five-column row:

| Reg. Clause | Requirement Summary | PDS Feature | Responsibility | PRD Reference |
|---|---|---|---|---|

Three things in it are worth more than the mapping itself:

- **The `Responsibility` column** classifies every clause as Software, Operator, or Hardware, so the
  document states plainly what software *cannot* solve. A separate **Responsibility Boundary
  Summary** (18 rows) exists specifically to support operator contracts and liability discussions.
- **Reg 13(4)** — a legal requirement that announcement audio occupy the 300–3000 Hz band — was
  discharged with **measurement, not assertion**. A Welch-PSD spectrum analysis of the rendered audio
  recorded 63.8% of total spectral energy inside the required band, with the method and tool versions
  written into the row.
- **The changelog is a quarter of the document.** It records nine revisions across three adversarial
  review rounds, including the corrections — one entry states that a row was *rewritten to drop the
  dishonest claim that the software prevents overlapping multi-tablet audio*. The document argues with
  itself in public, and loses.

---

## What is in this repository

| Path | What it is |
|---|---|
| [`living/STATE.md`](living/STATE.md) | The current-state snapshot, and a **45-entry decision ledger** of every standing decision still in force. Long — written as working memory, not as prose for a reader. |
| [`living/DESIGN-LANGUAGE.md`](living/DESIGN-LANGUAGE.md) | "Steelglass" — the product's design system, specified in tokens and enforced across both surfaces by binding repo rules. |
| [`sessions/`](sessions/) | **38 dated session records**, append-only, 2026-07-05 → 2026-07-30. One file per working session, each with a fixed schema: what changed, commits, decisions made, what was verified and on what hardware, what's next, what was deferred. |
| [`historical/`](historical/) | The frozen v3.9 planning documents: the compliance matrix above, the [PRD](historical/PRD.md) (~139 KB), the [Data Architecture](historical/Data-Architecture.md) (~182 KB), and a [narrative history](historical/PROJECT-HISTORY.md). Evidence and requirements context — **not** current authority. |

61 commits, May–July 2026. Roughly 106,000 words.

---

## The system, in brief

Two surfaces over one backend. The tablet is a display and stop-detection device; it authors nothing
and synthesises nothing.

**Android tablet** (Kotlin, Compose, Hilt, Room) — kiosk-mode, offline-first, GPS-driven. Detects
stops by per-stop geofence with an accuracy gate and a temporal-hysteresis confirmation layer, plays
pre-rendered audio, and drives a passenger display whose text is sized by physical measurement.
261 Kotlin source files, 50 test files.

**Next.js dashboard** — operators author routes against the full UK NaPTAN stop database, manage a
tablet fleet, and see journeys reported back. ~100 TypeScript source files.

**Supabase backend** — Postgres with **24 migrations** and **9 deployed Edge Functions**, including a
`pg_boss`-queued audio render pipeline that synthesises every announcement server-side. **All 8 tables
are row-level-security scoped**, multi-tenant from the first migration, with operator scope carried as
a JWT custom claim. **375,730 NaPTAN stops.**

Figures verified against the live production project on 2026-08-28, before decommissioning.

---

## A few decisions worth defending

Each of these is recorded with its reasoning in the ledger in [`living/STATE.md`](living/STATE.md).

- **Announcements fire on departure from the previous stop, not on arrival at the next.** Announcing
  on arrival tells passengers where they already are.
- **The visual half of a regulated announcement is co-equal with the audio.** The screen flash and
  text overlay fire even on a tablet with audio disabled, because the regulation mandates a combined
  alert and does not exempt silent units. Encoded as an architectural invariant, not a UI preference.
- **Regulated announcements are undroppable.** Announcement locking is deliberately asymmetric: a
  regulated announcement interrupts an informational one, an informational one never interrupts, and
  a legally-required announcement is never silently lost to a queue.
- **Passenger text is 22 mm physically, measured rather than assumed.** Font size is derived from a
  measured cap-height fraction against a screen calibrated with a bank card, because budget tablets
  misreport their own DPI. Long names scroll; they never shrink below the floor.
- **Audit the deployed shape before building against it.** Verifying what a function or schema
  *actually* looked like in production, rather than trusting the specification, repeatedly caught
  divergences — including an audio-path derivation defect that would have shipped broken.

---

## Notes for anyone reading this repository

**On the application code.** `pds-android` and `pds-dashboard` remain private. Publishing them
requires a proper secrets audit of their own git histories, which is a separate job; this repository
was always the better exhibit.

**On place names and route numbers.** The pilot was a personal contact at a bus operator, and their
route, stop and location data appeared throughout this repository. It has been replaced with a
fictional operator, and the repository's history was rewritten on 2026-07-30 to remove the originals
rather than merely overwriting them going forward. The substitutions are internally consistent, so
the documents still read correctly — but the places and route numbers you see are invented.
Handling a third party's commercial data carefully mattered more than a tidy history.

**On commit hashes in older files.** Because that history was rewritten, **short `pds-planning`
commit hashes quoted inside earlier session files and in the decision ledger refer to pre-rewrite
commits and no longer resolve.** Hashes cited for `pds-android` and `pds-dashboard` are unaffected —
those repositories were never rewritten.

**On the deployment.** The dashboard ran at `pds-dashboard.com` until the infrastructure was
decommissioned. It is written here in the past tense and deliberately never as a link — a lapsed
domain can be registered by anyone.

---

## The product, running

**[Watch the demo video](https://github.com/oliverjhj/pds-planning/releases/download/demo-video/pds-demo.mp4)** —
a five-minute walkthrough of the operator dashboard, a tablet pairing itself to a fleet, and the
tablet running a service, recorded against the live production system before it was decommissioned.
MP4, 92 MB; it downloads rather than streaming.

**[Read the case study](https://oliverjhj.github.io/pds-planning/)** — a one-page summary of what was
built, four decisions worth defending, and the evidence that it ran in production.
