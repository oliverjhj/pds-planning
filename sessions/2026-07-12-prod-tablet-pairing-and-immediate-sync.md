# Session: 2026-07-12 — Prod tablet pairing (env-mismatch + local.properties), empty-state link, immediate sync-on-pairing

**What changed:** First real end-to-end test of the tablet against the now-live production dashboard, plus two `pds-android` fixes surfaced by it. The owner disconnected the (staging-flavour) tablet and tried to pair it to `pds-dashboard.com`, getting "Something went wrong." Diagnosed by log evidence, then a second failure, then two code fixes — all glass-verified on the Lenovo (`prodDebug`) and pushed.

1. **Pairing failure #1 — environment mismatch (no code change).** Edge-function logs showed `generate-pairing-code` 200s on **production** but the tablet's `pair-device` calls landing on **staging** and returning **400** (timing correlated to the second: code minted on prod, redeemed against staging ~8–10 s later). Root cause: the installed tablet was the **staging** build flavour (hardwired to the staging Supabase project), so it could never find a production-minted code. Fix = install the **`prod`** flavour (`prodDebug`).

2. **Pairing failure #2 — empty prod keys in `local.properties`.** After building `prodDebug` the tablet reported "no internet connection" even though connected. Cause: `local.properties` had the prod *key names* but **blank values** (`SUPABASE_URL_PROD=` / `SUPABASE_PUBLISHABLE_KEY_PROD=`), and the build resolves missing keys to empty strings, so the client hit an empty URL → network failure surfaced as "no internet." Populated both from the prod project (`https://qopkjsihmdplmsoqrzji.supabase.co` + the `sb_publishable_...` key fetched via the read-only prod MCP). Rebuilt `prodDebug` → paired cleanly. (`local.properties` is git-ignored; not committed.)

3. **Empty-state dashboard link (`8434171`).** The first-run "no routes" empty state showed the old Vercel preview URL (`pds-dashboard-lovat.vercel.app`); repointed the `route_list_empty_dashboard_url` string to `https://pds-dashboard.com`. Reference text only (kiosk, not tappable).

4. **Immediate sync + FCM registration on pairing (`1377ac7`).** A freshly paired tablet showed an empty route list and got no hands-free pushes until a cold relaunch. Cause: `PairingScreen` self-navigated `PAIRING → ROUTE_LIST` inside the still-`Unpaired` NavHost, so `RootViewModel` never flipped to `Paired` and its paired-launch path (launch recovery + FCM token registration + sync-on-launch) never ran post-pairing. Fix (uniform, Option B chosen over a sync-only minimal patch): extracted that path into a guarded `enterPaired()`, reachable from both cold-boot `init` and a new public `onPaired()`; `PdsApp`'s pairing `onPaired` now calls `rootViewModel.onPaired()` (state flip swaps the NavHost onto the route list) instead of self-navigating. The run-once guard resets on `RequiresRepair` so a re-pair after a credential wipe re-runs the full path too. Fixes both the empty first sync **and** the latent "no pushes until first relaunch."

**Commits:**
- `pds-android` `8434171` — fix: point route-list empty-state link at pds-dashboard.com
- `pds-android` `1377ac7` — fix(FR-AT-01): sync + register FCM immediately on pairing, no relaunch

(Both owner-reviewed and pushed to `origin/main` this session — `7534746..1377ac7`.)

**Decisions made:**
- **`prodDebug` is now the pilot/testing build.** The tablet runs the `prod` flavour against the live production stack (dashboard `pds-dashboard.com` + prod Supabase), so both halves share one backend. Staging is kept in reserve for schema/migration validation (project rule: never verify schema changes against prod). Only one flavour installs at a time (same `applicationId`; side-by-side deferred to Stage 4), so switching backends means a reinstall. Accepted trade-off: `prodDebug` writes test data (device rows, journey_summaries, accumulating anon auth users) into production — harmless pre-pilot, tidy before the pilot company onboards.
- **Post-pairing should behave exactly like a cold boot that finds the device paired.** Chose the uniform RootViewModel-driven fix over a minimal PairingViewModel sync-fire specifically because it also fixes FCM registration latency, not just the sync — one coherent change at the seam that already owns the paired/unpaired decision.

**Verified:** `compileProdDebugKotlin` + `testProdDebugUnitTest` green. Glass-verified on the Lenovo (`prodDebug`, serial `HA2DG5NK`): fresh pairing populates the route list with **no relaunch**; a dashboard route save auto-syncs to the tablet via FCM push with **no relaunch**; empty-state link reads `pds-dashboard.com`. Owner confirmed "all working perfectly."

**What's next:** Continue owner-led iterative testing of the live product on `prodDebug`. The one substantive build item still ahead is the payment gate (What's Left #7).

**Banked / open:** Nothing new banked. Production will slowly accumulate `prodDebug` test cruft (device/auth rows, journey_summaries) — worth a cleanup pass before the pilot operator onboards, but not blocking.
