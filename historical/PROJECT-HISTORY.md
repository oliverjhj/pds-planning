# PROJECT-HISTORY.md
# Passenger Display System (PDS) — Project History

**Historical record up to the July 2026 workflow migration. Not updated further.**

This document is the narrative history of the Passenger Display System: how it was planned, how it was built, and the lessons that shaped it. It is a snapshot, frozen at the point the project moved to its current Claude-Code-centric workflow. It deliberately records the *story and the load-bearing decisions* — not every technical detail, which is apparent from the codebase itself.

**If you are a Claude (or a person) reading this to understand the project today: this file is history, not current state.** Current state and remaining work live in `pds-planning/living/STATE.md`. Ongoing session-by-session history is in `pds-planning/sessions/`. The original planning documents (PRD, Data Architecture, Compliance Mapping Matrix) are alongside this file in `pds-planning/historical/` as the frozen requirements and compliance record. Where any of those and the current implementation disagree, the current implementation is authoritative.

---

## What PDS Is

PDS is a legally-regulated, two-surface SaaS product that provides compliant audio and visual passenger information on UK buses operating rail-replacement services. It consists of an **Android tablet application** (kiosk-mode, offline-first, GPS-driven) that displays and announces route information to passengers, and a **Next.js web dashboard** where bus operators author routes and manage their tablet fleet. Both share a single **Supabase** backend, with audio pre-rendered server-side via Google Cloud TTS. The product is regulated under the UK Public Service Vehicles (Accessible Information) Regulations 2023 (PSVAR); much of its design exists to satisfy those regulations. The pilot customer is a small bus company.

---

## The Arc

### Planning and adversarial review (April–May 2026)

An earlier attempt at this product was abandoned due to scope drift and disconnects between planning and implementation. PDS is a clean restart, built on revised planning documents and a deliberately tighter workflow. Planning produced the PRD, Data Architecture, Compliance Mapping Matrix, and an operating manual (WORKFLOW), plus a builder-facing conventions file per repository.

Before any code was written, the planning suite was put through three rounds of adversarial review — a reviewer looking for contradictions and gaps rather than fixing them in place. Round 1 surfaced ~19 structural findings; round 2 ~30 (including the realisation that the builder-convention files had drifted from the specs); round 3 ~20 mostly-fixable items plus one empirical question. The review was stopped deliberately after round 3 on the judgment that a fourth round would yield almost nothing. The documents were then frozen, and the discipline of "the specs are the contract; if a spec is wrong, that triggers re-planning, not a quiet edit" was adopted to prevent the scope-drift pathology that had killed the earlier attempt.

### Stage 1 — Dashboard MVP (May 2026)

The web dashboard was built first and deployed to production. It implemented operator signup and approval, route creation with NaPTAN stop search across the full UK dataset, per-stop proximity and segment-type configuration, return-route generation, device pairing, and the fleet view. Critically, it also built the **server-side audio render pipeline**: on route save, a job is queued and a worker renders each announcement to audio using a single locked voice, stores it at version-keyed paths, and — only on successful render — pushes a notification so tablets sync. This "render-then-notify" ordering ensures a tablet is never told a route is ready before its audio actually is. Google Cloud TTS, Firebase Cloud Messaging, and Supabase Storage were all wired and proven end-to-end.

### Stage 2 — Android foundation, sync, and recovery (late May–June 2026)

The tablet application was built in phases.

**Foundation** established the toolchain and the trust-sensitive machinery: first-run pairing via a 6-digit code, encrypted on-device credential storage, and FCM registration. A deliberate security decision replaced Android's deprecated encrypted-preferences library with a Keystore-wrapped encrypted datastore, and device credentials were kept in a single store so they could be wiped atomically.

**The sync engine** followed: an ordered, single-flight sequence that reads operator status, downloads routes and stops, downloads route audio, and is driven by three triggers (a push on route change, a network-reconnect trigger, and a periodic fallback). This phase produced the first fully hands-free propagation of a dashboard route edit to the tablet with no human intervention. It also produced one of the project's most valuable catches (see *Deployed-shape audits* below): the audio Storage path segment could not be derived the way the spec described, because the value it relied on moved on every row update — the deployed system had evolved a stable render-stamped value instead, and the tablet was corrected to use it verbatim.

**Recovery and liveness** completed the foundation: a reactive JWT-refresh path that silently re-mints the device's token on expiry without re-pairing (which removed a roughly one-hour test constraint that had shaped earlier sessions), and a lifecycle-bound heartbeat that keeps the device's last-seen timestamp current, deliberately kept independent of sync.

### The re-sequencing into vertical slices (June 2026)

With only the user interface and journey runtime left, the remaining lettered phases were re-grouped into four vertical slices, each ending in something testable on the tablet. This was explicitly a change of ceremony, not scope: every requirement and the frozen spec stayed intact — the back half of the product is the compliance-critical half, so loosening requirements was exactly what was *not* done.

### The passenger surface, then the journey runtime (June 2026)

The routes UI, journey-start gating (including the check that a route's audio is present before a journey can begin), and the **passenger-display compliance cluster** were built first: screen calibration against a bank card, physically-measured ≥22mm text that resists system font-scaling, the next-stop hero region with a no-truncation fit-to-floor layout, and the stylised tube-map progress view.

Then came the **journey-runtime compliance core** — the meatiest and most compliance-sensitive work in the project. GPS-driven stop detection with a strict-sequential state machine; audio playback of the pre-rendered files; the render of the bundled fixed-announcement audio; the announcement primitives (alert chime, screen flash, text overlay); journey completion and termination; and hail-and-ride section handling. This is where the load-bearing compliance rules below were established and hard-won on real hardware.

### Diversions, then Slice 4 (June–July 2026)

Driver diversion handling was built in three slices: the author/observe surface (the driver marks stops as skipped and hears the regulated diversion announcements), the GPS auto-skip loop (the journey silently advances over skipped stops), and replay-on-resume (a diversion announcement is re-played once if the app restarts mid-diversion). The auto-skip slice was the only work to touch the verified detection core and was treated as the highest-risk, most-isolated change accordingly.

Slice 4 completed the feature set: Level 1 kiosk mode (screen pinning plus default-launcher registration, host-owned across the whole app), the admin PIN and the full admin menu (calibration, force-sync, deregister, volume floor, auto-timeout, event-log viewer), and the post-journey summary upload of anonymous count metrics to the backend. With that, the product's feature set was functionally complete on both surfaces.

### The workflow migration (July 2026)

At this point the project shifted its working method. During construction, planning had been done in a separate "architect" chat that read the documents and wrote scoped build prompts, while a separate builder executed and committed — a separation that repeatedly caught real problems while there was a lot to get wrong. With the product largely built and the remaining work being verification, hardening, and polish, that separation's value had diminished relative to being able to read the actual code directly. The project consolidated into a single Claude Code workspace rooted above all repositories, with a living state file and an append-only session log replacing the previously hand-maintained monolithic context document. This document is the historical record produced at that migration.

A short pre-migration verification pass (a requirement-coverage sweep and two code recons) confirmed the feature set was complete bar one coherent remaining slice — driver-facing volume/mute controls, Bluetooth audio-output handling, and a manual hail-and-ride fallback — plus backend error telemetry, a production-shape audit, and a full-app GPS verification run. Those are carried in `living/STATE.md`, not here.

---

## Load-Bearing Lessons

These are the few decisions and disciplines that shaped the whole project and are worth carrying forward.

**The compliance rules that recur everywhere.** Three rules proved load-bearing across the entire announcement layer and every future change inherits them: the **co-equal visual rule** — the screen flash and text overlay for a regulated announcement fire regardless of whether audio is enabled, because they are the visual half of a legally-mandated combined alert and a silent tablet must still show them; the **asymmetric, undroppable locking rule** — a regulated announcement interrupts a non-regulated one rather than ever being silently dropped, because losing a legally-required announcement to an in-flight informational one is a compliance failure; and the **event-index-versus-approached-index discipline** — the stop being *announced/passed* and the stop being *approached* are deliberately different indices, and audio and boundary logic must key on the approached index while event logging keys on the announced one. This last one was the single most recurring trap of the runtime work. Alongside these sits the **≥22mm physical text guarantee** — passenger text is sized by physical measurement and never truncates below the legal floor; it grows line count instead.

**Deployed-shape audits before touching deployed surfaces.** A recurring discipline: before building against any deployed backend function, RPC, or schema, audit its actual deployed shape rather than trusting the spec. This caught real, shipped-defect-preventing divergences more than once — most valuably the audio-path-derivation defect, where the spec's described mechanism would have produced broken paths on live data. "Assume nothing about the deployed system; verify it" earned its place as standing practice.

**The architect/builder separation, and knowing when it had done its job.** For most of construction, a planning role that read everything and reasoned about the whole, kept separate from a builder that executed, was genuinely valuable — it caught design problems before they were committed, and plan-mode review functioned as a real two-way gate, occasionally overturning the architect on sound reasoning. Recognising when that separation's value had diminished — once the work became verification and polish and reading the real code mattered more than a separate planning layer — is what prompted the July 2026 migration.

**Instrument before theorising.** Repeatedly, when on-device behaviour surprised us, the fast path to the truth was to add a diagnostic log and read the actual value, not to reason about what "must" be happening. Several long debugging detours came from inferring instead of measuring; the lesson was cheap instrumentation over expensive theory.

**Reg 13(4) was empirically verified, not just designed for.** The alert chime that precedes regulated announcements must fall within the audible frequency band the regulation requires. Rather than assume the rendered chime satisfied this, its audio was spectrally analysed: the chime's energy peaks at approximately 1600 Hz with essentially all of it inside the required 300–3000 Hz band. This is recorded here because it is one of the few places a compliance claim was *measured and confirmed* rather than asserted — it is evidence, not intent.

---

## Closing Note

This history ends at the July 2026 workflow migration and is not maintained beyond it. For anything about the project *now* — what is built, what remains, what is being worked on, and the current decision ledger — read `pds-planning/living/STATE.md`. For the record of what happened after this snapshot, read the dated entries in `pds-planning/sessions/`. For the original requirements and compliance mapping, read the frozen documents alongside this one in `pds-planning/historical/`.
