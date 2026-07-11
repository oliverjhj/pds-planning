# Session: 2026-07-11 — The Booking Hall visual redesign, end to end

**What changed:** The complete visual redesign of both surfaces, from first exploration to shipped, glass-verified code and a codified design system — all in one session. The arc: (1) a small docs fix discharged the last Bank-and-Assess nit; (2) Anthropic's `frontend-design` skill was installed at the workspace `.claude/skills/` to counter generic AI styling; (3) a five-round owner-driven design exploration ran as claude.ai artifact galleries — 10 directions → 20 heritage/green directions → 10 hybrids in both modes → 10 colour-focused options on the four survivors → 5 dark-mode candidates — converging on **"Booking Hall"**: Gelasio serif, ticket-stock dashed edges, rubber-stamp statuses, a crimson serial Nº, light parchment `#FAF6EA`/`#2F5D43`, dark "Wheat" `#0D2A1C`/`#E6DDBC`/celadon `#A9C7AD`, and a half-light/half-dark circle toggle on both surfaces; (4) a plan-mode implementation plan (owner-approved) was executed as **15 commits** — 7 dashboard, 8 Android; (5) the design system was codified three ways: the `pds-design-language` skill (workspace `.claude/skills/`), the "PDS Booking Hall" Claude Design project (12 registered component cards), and `living/DESIGN-LANGUAGE.md` (reference copy, added at this close-out).

**Compliance findings along the way:** Reg 14(4)'s ≥22mm floor covers the **route name** as well as the next stop (already met in code — route name sits on the floor); Reg 14(5)(a) bans all-caps on the passenger display, so the redesign's letterspaced-caps idiom is dashboard/driver-side only; the announcement flash/overlay restyle is **colors only** — `RegulatedAnnouncementCoordinator` timing untouched; and the serif's 22mm honesty is kept by measuring cap height from the bundled Gelasio faces (minimum across the four weights) with the measured face baked into the rendered style.

**Commits:**
- pds-android `cc0b507` — docs: journey_events lifecycle is a 30-day retention prune, not clear-on-start
- pds-dashboard `a94cb77` — feat: Gelasio + Courier Prime via next/font
- pds-dashboard `6265d9d` — feat: Booking Hall light + Wheat dark token palettes
- pds-dashboard `0577005` — feat: theme toggle with cookie persistence (server-side read, zero FOUC)
- pds-dashboard `6638e6f` — feat: signature graphics (ticket edges, stamps, serial Nº, wordmark)
- pds-dashboard `8fae667` — feat: table + badge restyle, hail-and-ride in ephemera red
- pds-dashboard `785dce3` — refactor: raw color literals → semantic tokens (amber stays amber)
- pds-dashboard `6a5120e` — docs: CLAUDE.md Rule 16 (Booking Hall token discipline)
- pds-android `2c2aed2` — chore: bundle Gelasio (4 static weights, OFL) + driver/admin typography
- pds-android `6580b7a` — feat: serif through PhysicalText with measured cap height (Rule 13/FR-AT-34)
- pds-android `f7b4ca1` — refactor: PassengerPalette CompositionLocal (zero visual change)
- pds-android `b041e7d` — feat: Wheat passenger palette + announcement inversion colors + PassengerPaletteTest
- pds-android `92f27bb` — feat: Booking Hall M3 schemes, PDSTheme(darkTheme), PassengerLight
- pds-android `2460b69` — feat: DisplayThemeStore store-triplet + observe/set use cases
- pds-android `5b3c9fc` — feat: driver-panel theme toggle + MainActivity wiring (Rule 20)
- pds-android `2ec891e` — docs: CLAUDE.md Rule 20 + Gelasio cap-height note
- pds-planning (this commit) — session log, STATE.md close-out, `living/DESIGN-LANGUAGE.md`

(All code commits reviewed and pushed by the owner.)

**Decisions made:**
- **Booking Hall is the product's visual identity** (owner sign-off after five artifact rounds; ledger #25). Font = Gelasio (OFL, Georgia-metric companion — Georgia itself isn't bundleable). Tablet default = dark Wheat (preserves the historical always-dark passenger posture); dashboard default = light; both toggles one-tap, no menu.
- **The passenger region's "always dark, not theme-switchable" posture is superseded** — deliberately, owner-approved: passenger colors now flow through `LocalPassengerPalette` (Wheat/Light), switched together with the M3 scheme by the driver-controlled `DisplayThemeStore` (never system/OS-derived; kiosk uniformity kept per setting).
- **The announcement flash/overlay became the palette inversion** (wheat-on-green dark / green-on-cream light) — colors only; coordinator timing, chime simultaneity, and audio-independence untouched. `flashFill` is the single tuning point if salience ever needs raising.
- **Amber is semantic in every theme** — regression-guarded by `PassengerPaletteTest`'s amber-band assertion (plus WCAG contrast floors: ink/ground and overlay pairs ≥7:1 in both palettes).
- **New repo rules:** dashboard Rule 16 (token-only colors, amber never rebranded, generated ui/ untouched), Android Rule 20 (driver-controlled theme, palette-only passenger colors, no passenger caps, coordinator owns announcement timing).

**Verified:** Dashboard — build/typecheck/lint/vitest per commit + a Playwright pass in both modes (exact hexes confirmed on computed styles, server-side `.dark` with no flash, contrast measured 7.02:1 light / 11.31:1 dark, screenshots reviewed). Android — assembleDebug + full unit suite per commit (incl. the new palette-contrast and default-dark tests) + grep gates (zero uppercase transforms on passenger surfaces, zero orphaned constants). **Owner glass-verified the full checklist on the real tablet — passed** ("on glass looks perfect"): 22mm serif caps both themes, inversion flash salience on the regulated types, one-tap whole-UI toggle with persistence, amber markers, mixed case, tube-map states.

**What's next:** The pilot — unchanged. The product now enters it wearing its own face.

**Banked / open:**
- The workspace `pds-design-language` skill and `living/DESIGN-LANGUAGE.md` are manually-synced copies — if the design ever changes, update both (the skill is what Claude loads; the repo rules 16/20 outrank both).
- Claude Design project "PDS Booking Hall" (12 cards) exists under the owner's claude.ai login; card sources regenerate from the session scratchpad's `gen_ds.py` pattern if it ever needs rebuilding.
- Anthropic's `frontend-design` skill remains installed at the workspace for future divergent design work.
- Everything else unchanged (accepted postures only).
