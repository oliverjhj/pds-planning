# PDS Design Language — "Steelglass"

> **Provenance:** reference copy of the `pds-design-language` skill (authoritative working copy:
> workspace `.claude/skills/pds-design-language/SKILL.md` — kept in sync manually; if the two ever
> diverge, the skill is what Claude actually loads, so fix the drift). Owner-signed-off and
> glass-verified 2026-07-17. Living because the design language is current authority, not history.

The identity: strictly neutral monochrome, machined glass. A near-black or pure-white canvas gradient, translucent glass panels with hairline borders and an inset top-highlight, Inter for UI, JetBrains Mono for data, and colour used only with intent — green means ready, amber means warning, red means failure or a terminal action. Every UI addition to either repo must read as part of this system — never default-Material, never default-shadcn, never decorative colour. (Supersedes "Booking Hall", 2026-07-17.)

**Browsable reference:** the "PDS Steelglass" project at claude.ai/design (14 rendered cards). **Binding repo rules:** pds-dashboard `CLAUDE.md` Rule 16, pds-android `CLAUDE.md` Rule 20 — where this file and a repo rule differ, the repo rule wins.

## Tokens

| Role | Dark "D2 near-black" (DEFAULT, both surfaces) | Light "L1 pure-white" |
|---|---|---|
| Canvas gradient (160°) | `#0A0A0A → #151515` | `#FFFFFF → #F6F6F6` |
| Ground anchor (worst-contrast gradient end) | `#151515` | `#F6F6F6` |
| Ink / body text | `#EAEAEA` | `#1C1C1C` |
| Muted text | `#8F8F8F` | `#737373` |
| Accent (tablet "Next stop" caption) | `#C9C9C9` | `#5C5C5C` |
| Glass panel | white @ 5% + blur (web) | white @ 62% + blur (web) |
| Border hairline | white @ 10% | black @ 10% |
| Top highlight (sheen) | inset 1px white @ 7% | inset 1px white @ 70–85% |
| Primary action | `#EAEAEA` fill, `#111111` text | `#1C1C1C` fill, `#FFFFFF` text |
| OK / ready | `#4ADE80` | `#178344` |
| Amber — SEMANTIC warnings only | `#FBBF24` | `#B45309` (hue watch-item: tune UP only) |
| Failure / destructive | `#F87171` | `#C02626` |
| Ephemera (terminal crimson — End Journey) | `#E5484D` | `#B3382E` |
| Flash / overlay (tablet) | `#F0F0F0` ground, `#161616` ink (inversion) | `#161616` ground, `#F0F0F0` ink |
| Tube-map (tablet) | completed `#4A4A4A` · upcoming `#8F8F8F` · highlight `#EAEAEA` | `#C7C7C7` · `#6B6B6B` · `#1C1C1C` |
| Radius | 12px panels · 8–9px controls · 999px chips | same |

**Never:** decorative colour of any kind (the palette is greys + the four semantic roles); amber used as a button/control accent (it is warnings-only, plus the engaged-mute fill); new raw hex values on branded surfaces — extend the token systems instead (dashboard: `globals.css` custom properties + `@theme` mapping; Android: a new `PassengerPalette` role defined in BOTH palettes + a `PassengerPaletteTest` assertion). The Android palette's `background` role is always the gradient's worst-contrast end, so contrast holds at every gradient point.

## Type

- **Inter** (OFL) everywhere, static weights 400/500/600/700. Dashboard: loaded via `next/font` onto `--font-sans`/`--font-heading`. Android: `res/font/inter_*.ttf` via `PdsSans` in `Type.kt` — static per-weight TTFs, never the variable font (the 22 mm cap-height measurement takes the minimum across the bundled weight resources).
- **JetBrains Mono** (OFL) is the data voice — serials, versions, times, NaPTAN codes, route numbers, IDs — never body text. Dashboard: `--font-geist-mono` via `next/font`. Android: `res/font/jetbrains_mono_regular.ttf` via `PdsMono`.
- Passenger text on the tablet sizes ONLY through `rememberPhysicalTextStyle` (22mm floor, cap height measured from Inter). Never a raw `sp` on a passenger-facing string.

## Signature elements (use them; don't invent parallel idioms)

- **Canvas gradient** — every screen floats on the 160° wash. Dashboard: painted on `body` in `@layer base` (fixed attachment). Android: `Modifier.steelglassCanvas(palette)` from `ui/theme/Steelglass.kt` — painted once at MainActivity's root for Material screens (their Scaffolds are containerColor-transparent) and at the passenger/journey roots. Never hand-roll the gradient.
- **glass-panel** — the translucent machined panel: card ground + 1px hairline border + inset top-highlight + backdrop blur (blur is web-only progressive enhancement; Android uses translucency without blur by design — no blur dependency). Dashboard utility class `glass-panel` (`--card` is translucent, so shadcn Cards are glass by construction). Android: `SteelglassPanel` in `ui/theme/Steelglass.kt` for decorative in-screen panels only — full-screen driver overlays keep the near-opaque `panelSurface`, and Material dialogs/sheets stay opaque scheme surfaces.
- **chip** — status pills: 1px hairline border in `currentColor`, full radius, letterspaced caps. Dashboard utility class `chip`. **Dashboard/driver-side only** (passenger surfaces stay mixed case).
- **Driver-panel buttons (tablet)** — SOLID filled, no border: every driver-control-panel button is a flat, small-radius filled button so the panel reads as one uniform block. **Ink fill + ground text** (the ground↔ink inversion: light-on-near-black dark, near-black-on-light light) for every control, with two solid-filled exceptions: *terminal* (End Journey) = **ephemera crimson**, and *Emergency-Mute-engaged* = **semantic amber** (the muted-state indicator). Amber is NEVER a generic button fill — only engaged-mute + the GPS-lost / speaker-disconnect markers. No dashed / ghost / outline on the driver panel; dashes in the passenger region are `TubeMapView`'s hail-and-ride semantics only. Enforced by Android Rule 20.
- **Mono data voice** — anywhere a serial, version, timestamp, count, or route number appears, it renders in the mono face, usually muted. This replaces the retired serial-Nº ornament: real identifiers only, no decorative digits.
- **Product mark — the "pinned screen"** — the passenger-display rectangle with a location pin knocked out of it. Strictly monochrome in Steelglass polarity: light `#EAEAEA` mark on a `#111111` ground, never recoloured and never given a second colour. Two deliberate forms of ONE identity: the **full mark** (rect + knocked-out pin) is the identity wherever it has room — the Android adaptive launcher (`ic_launcher_foreground.xml`, a single **even-odd** path so the pin hole shows the ground through and the monochrome themed-icon layer inherits the knockout for free; sized to the 66dp safe zone) and future branding surfaces; the **favicon carries the pin element alone** on the `#111111` disc with a faint hairline ring (`src/app/icon.svg`) — a tab-slot fit decision taken after the full rect silhouette sat visibly high in the tab, **not** a second identity. Keep the disc treatment; don't recreate the legacy density launcher webps (deleted — unreachable at minSdk 26).
- **Wordmark** — "PDS" bold, `.14em` tracking. **Dashboard only** — the tablet's banner shows the operator's company name in mixed case instead (Reg 14(5)(a)).
- **Journey banner band** (tablet) — a 56dp header strip on the passenger display: operator company name left (Inter, mixed case), solid 1px `glassBorder` bottom hairline. It hosts the top-end control cluster and is what lets the regulated ticker lines below run full width; dropped at the smallest layout tier. **The band carries no route identifier** — the route designation leads the regulated route line directly below it, and showing it in both places was the same value twice, two rows apart (2026-07-18).
- **Mode toggle** — the half-light/half-dark circle (inline SVG on web, Canvas `drawArc` on Android). Dashboard header (cookie `pds-theme`, read server-side — never a pre-hydration script; **absent cookie = dark**) and the tablet's app-wide top-end control cluster beside the admin gear (`presentation/common/ThemeToggleButton`; `DisplayThemeStore`, default dark). Cluster chips draw from the Material scheme — never raw black/white constants.

## Interaction & affordance

Clickable things look clickable, respond when used, and inactive things don't. This is the principle — the cursor rule below is one worked example of it, not the whole of it.

- **Affordance & feedback.** Every interactive control advertises itself and responds. On the dashboard that is a pointer cursor from **one base-layer rule** in `globals.css` (covering `button`, `[role="switch"]`, radio/checkbox and their wrapping labels — never patched per-component; dashboard Rule 16) plus hover/pressed feedback; on the tablet it is a generous touch target and the driver panel's brief press confirmation. A CSS rule can only reach what the browser already knows is interactive, so **a clickable `<div>` with no `<button>` and no `role` is invisible to it *and* to assistive tech** — make such a row a real `<button>`/`<a>`, or give it `role="button"` + `tabIndex={0}` + a key handler, so affordance and accessibility hold together.
- **Focus-visible.** Keyboard focus is always visible — a focus ring on every dashboard control (shadcn's `focus-visible:ring`, never stripped). A pointer-and-keyboard concern only: the tablet is a touch kiosk with no focus traversal.
- **Disabled never advertises a click.** A disabled control drops the pointer cursor (the base rule excludes `:disabled` / `[data-disabled]`) and reads as plainly inactive, never as enabled. On the tablet's driver panel, disabled buttons are the **same solid fill, faded** — never a border or grey swap (Android Rule 20).

## Compliance constraints the design must never break (tablet passenger surface)

1. **Mixed case only** — Reg 14(5)(a). No `text-transform: uppercase`, no `uppercase()`, no small caps on anything a passenger reads. Caps live on the dashboard.
2. **≥22mm physical text** — Reg 14(4). Route name, stop name, announcement text all go through the PhysicalText machinery; never truncate, grow line count instead.
3. **Contrast** — Reg 14(5)(b). Ink/ground ≥7:1 at BOTH gradient ends (both palettes are; `PassengerPaletteTest` guards `background` and `backgroundDeep`).
4. **Amber is semantic** — GPS-lost, speaker-disconnect, and mute markers are amber in every theme.
5. **Announcement flash/overlay** — colors come from the palette (`flashFill`, `overlayBackground/-OnBackground`, always alpha-1); the 500ms/8s timing, chime simultaneity, and audio-independence belong to `RegulatedAnnouncementCoordinator` and are untouchable from design work.
6. Passenger colors flow only through `LocalPassengerPalette` — never new top-level constants; Steelglass surfaces flow only through the `Steelglass.kt` helpers.

## Voice

Plain verbs, sentence case, specific over clever ("Save & render audio", "Rendered", "Contact your fleet manager…"). Errors say what happened and what to do. The register is a precision instrument: calm, exact, never cute.

## Sync procedure — the design system lives in three places

**Any change to the design language — tokens/colours, type, signature elements, component idioms, control placement, voice — updates all three copies in the same session:**

1. **The skill** — workspace `.claude/skills/pds-design-language/SKILL.md` (the working copy Claude loads).
2. **`pds-planning/living/DESIGN-LANGUAGE.md`** — the reference copy; body kept word-for-word identical to the skill (only the skill frontmatter / provenance header differ).
3. **The "PDS Steelglass" Claude Design project** (claude.ai/design, projectId `89aaa8cf-5142-44e3-ae7b-18e7b34ed8b3`) — update the affected card(s) with the DesignSync tool: `list_files` → `get_file` on the affected cards → `finalize_plan` → `write_files`. Cards are self-contained HTML sharing a common inline style block, each with a first-line `<!-- @dsCard group="…" -->` marker (keep it; no re-registration needed).

A design change is not done until all three match. If the change touches what a binding repo rule mandates (dashboard Rule 16 / Android Rule 20), that rule is updated too, via the normal commit flow — the repo rules outrank all three copies.
