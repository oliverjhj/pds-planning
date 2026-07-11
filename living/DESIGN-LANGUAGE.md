# PDS Design Language — "Booking Hall"

> **Provenance:** reference copy of the `pds-design-language` skill (authoritative working copy:
> workspace `.claude/skills/pds-design-language/SKILL.md` — kept in sync manually; if the two ever
> diverge, the skill is what Claude actually loads, so fix the drift). Owner-signed-off and
> glass-verified 2026-07-11. Living because the design language is current authority, not history.

The identity: an Edwardian railway booking hall. Gelasio serif, ticket-stock dashed edges, rubber-stamp statuses, a crimson serial Nº, parchment by day and deep-green "Wheat" by night. Every UI addition to either repo must read as part of this system — never default-Material, never default-shadcn, never Inter-on-white.

**Browsable reference:** the "PDS Booking Hall" project at claude.ai/design (12 rendered cards). **Binding repo rules:** pds-dashboard `CLAUDE.md` Rule 16, pds-android `CLAUDE.md` Rule 20 — where this file and a repo rule differ, the repo rule wins.

## Tokens

| Role | Light (default: dashboard) | Wheat dark (default: tablet) |
|---|---|---|
| Ground | `#FAF6EA` | `#0D2A1C` |
| Ink / body text | `#2F5D43` | `#E6DDBC` (wheat) |
| Primary interactive | `#2F5D43` | `#A9C7AD` (celadon) |
| Card / raised | `#FDFAF1` | `#123524` |
| Table-header tint | `#F1ECD8` | `#123524` |
| Muted text | `#79836F` | `#93A487` |
| Ephemera (serial, stamps, H&R, destructive) | `#8C2F28` | `#CF6A55` |
| Amber — SEMANTIC warnings only | `#8A5A00` on `#F2E6C8` | `#D9A441` on `#33290F` |
| Borders | ink @ 32% alpha | wheat @ 28% alpha |
| Flash / overlay (tablet) | deep-green ground, cream ink | wheat ground, deep-green ink (inversion) |
| Tube-map (tablet) | completed `#B3BFAC` · upcoming `#79836F` · highlight `#2F5D43` | `#5C7260` · `#93A487` · `#A9C7AD` |

**Never:** body text in gold/celadon-only; amber used decoratively; new raw hex values on branded surfaces — extend the token systems instead (dashboard: `globals.css` custom properties + `@theme` mapping; Android: a new `PassengerPalette` role defined in BOTH palettes + a `PassengerPaletteTest` assertion).

## Type

- **Gelasio** (OFL) everywhere, weights 400/500/600/700. Dashboard: loaded via `next/font` onto `--font-sans`/`--font-heading` (+ italics). Android: `res/font/gelasio_*.ttf` via `PdsSerif` in `Type.kt`.
- **Mono** (Courier Prime on web; platform mono on Android) for serials, NaPTAN codes, IDs — never body text.
- Passenger text on the tablet sizes ONLY through `rememberPhysicalTextStyle` (22mm floor, cap height measured from Gelasio). Never a raw `sp` on a passenger-facing string.

## Signature elements (use them; don't invent parallel idioms)

- **ticket-edge** — 1.5px dashed border; tables, cards, panels, ghost buttons. Dashboard utility class `ticket-edge`.
- **stamp** — status badges: 3px double border, −3° rotation, letterspaced caps. Dashboard utility class `stamp`. **Dashboard/driver-side only.**
- **Serial Nº** — decorative mono "Nº ####" in ephemera red. Dashboard: digits derived from the entity UUID (`SerialNo` component), detail-page headers. Tablet: the route's operator-assigned number on the journey banner. On Android the red is the `ephemera` `PassengerPalette` role (both palettes, `PassengerPaletteTest`-guarded: ≥3:1 on the ground, hue outside the amber band).
- **Wordmark** — "PDS" bold caps, `.14em` tracking. **Dashboard only** — the tablet's banner shows the operator's company name in mixed case instead (Reg 14(5)(a)).
- **Journey banner band** (tablet, 2026-07-11) — a 56dp header strip on the passenger display: operator company name left (Gelasio, mixed case), route-number Nº right (ephemera mono), dashed ticket-edge bottom border (ink @ ~30%). It hosts the top-end control cluster and is what lets the regulated ticker lines below run full width; dropped at the smallest layout tier.
- **Mode toggle** — the half-light/half-dark circle (inline SVG on web, Canvas `drawArc` on Android). Dashboard header (cookie `pds-theme`, read server-side — never a pre-hydration script) and, since 2026-07-11, the tablet's **app-wide top-end control cluster** beside the admin gear (`presentation/common/ThemeToggleButton`; `DisplayThemeStore`, default dark) — no longer in the driver-panel header. Cluster chips draw from the Material scheme (surfaceContainerHigh disc, outlineVariant ring, primary glyphs) — never raw black/white.
- Radius stays small (0.25rem web / 2–4dp feel) — ticket stock, not bubbles.

## Compliance constraints the design must never break (tablet passenger surface)

1. **Mixed case only** — Reg 14(5)(a). No `text-transform: uppercase`, no `uppercase()`, no small caps on anything a passenger reads. Caps live on the dashboard.
2. **≥22mm physical text** — Reg 14(4). Route name, stop name, announcement text all go through the PhysicalText machinery; never truncate, grow line count instead.
3. **Contrast** — Reg 14(5)(b). Ink/ground ≥7:1 (both palettes are; `PassengerPaletteTest` guards it).
4. **Amber is semantic** — GPS-lost, speaker-disconnect, and mute markers are amber in every theme.
5. **Announcement flash/overlay** — colors come from the palette (`flashFill`, `overlayBackground/-OnBackground`); the 500ms/8s timing, chime simultaneity, and audio-independence belong to `RegulatedAnnouncementCoordinator` and are untouchable from design work.
6. Passenger colors flow only through `LocalPassengerPalette` — never new top-level constants.

## Voice

Plain verbs, sentence case, specific over clever ("Save & render audio", "Rendered", "Contact your fleet manager…"). Errors say what happened and what to do. The register is a competent booking clerk: precise, unhurried, never cute.
