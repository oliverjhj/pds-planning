# Session: 2026-07-18 — Product logo: the "pinned screen" mark (third session)

**What changed:** The product got its logo. A 50-option black-and-white logo exploration artifact (three directions: clean display marks / eye-characters in the owner's reference style / hand-drawn pen-and-ink; each option shown with an Android-circle and true-16px favicon preview) was built and reviewed by the owner, who picked **A13 "Pinned Screen"** — the passenger display rectangle with a location pin knocked out of it. Implemented as the Android adaptive launcher icon (single even-odd path, `#EAEAEA` on the existing `#111111` ground, pin is a true knockout so the monochrome themed-icon layer inherits it; sized to the 66dp safe zone) and as the dashboard favicon. The favicon went through one owner-driven revision: the screen-rect silhouette sat visibly high in the browser tab slot, so the shipped favicon is the **pin alone on the `#111111` circle disc** (same disc + faint hairline treatment as the old ring-and-bar favicon), optically centred. Owner ruling: the full pinned-screen mark is the main product logo (Android launcher and future branding); the circle-pin is the favicon-fit variant only. The ten dead template-era density webps (`ic_launcher*.webp`, mdpi–xxxhdpi — unreachable with minSdk 26 since the anydpi adaptive XMLs always win) were removed on owner instruction. Everything pushed at owner request; both Vercel deploys confirmed (webhook fired both times — no repeat of the 2026-07-17 miss) and the live favicon byte-verified on `pds-dashboard.com/icon.svg` by curl.

**Commits:**
- pds-android `c80ea15` — chore: pinned-screen launcher mark (owner-picked A13)
- pds-android `4a8ecd4` — chore: remove dead legacy launcher webps
- pds-dashboard `9d12a88` — chore: pinned-screen favicon (owner-picked A13)
- pds-dashboard `5c9a6b8` — fix: favicon — pin in a circle, optically centred

**Decisions made:**
- **The A13 "pinned screen" is the product mark** (new ledger #35; supersedes ledger #32's identity-resources line: the ticker-bar-in-ring launcher mark and ring-and-bar favicon are retired). The favicon deliberately carries only the pin-in-circle element — a tab-slot fit decision, not a second identity.
- Trademark/copyright question (owner asked re Google Maps' pin): assessed as low-risk — the mark is original geometry, the generic teardrop pin is ubiquitous, and trademark confusion requires similar mark + market + usage, none of which apply. **A quick UK IPO trademark screen added to the Phase 4 legal work** as standard pre-launch hygiene. Not legal advice; owner may fold it into the Phase 4 legal review.
- Webp cleanup done as its own scoped commit only after the owner asked (no-opportunistic-refactor rule held; flagged first, cleaned second).

**Verified:** Android resource processing green after both icon commits (`:app:processStagingDebugResources`); dashboard `npm run build` green twice; Vercel production deployments `dpl_5LUNRt…` (9d12a88) and `dpl_5G1ZXb…` (5c9a6b8) both READY and aliased to `pds-dashboard.com`; live `icon.svg` fetched from the domain and byte-matched the committed file. **Not yet on glass:** the launcher icon on the Lenovo — pending the next tablet install (same pass as the `e20fe6d` TopAppBar check).

**What's next:** Unchanged — the tube-map transition visual (Phase 2's last named item).

**Banked / open:**
- Launcher-icon glass look at next install (added to the standing pending-glass-check item).
- The logo/mark is not yet documented in the three design-system copies (DESIGN-LANGUAGE.md / skill / Claude Design cards have no logo section) — flagged, not silently skipped; add a mark section in a future design session if the identity work continues (ledger #27 sync applies when it does).
- The logo exploration artifact ("PDS Logo Exploration", claude.ai/code/artifact/2e3585f6-dfe4-4ace-9058-c5d59acc5a9c) holds the other 49 options if branding work ever wants alternates/companions (the eye-character family in particular).
- Browser-side favicon caching means the owner's own tab may lag the deployed icon; server verified correct.
