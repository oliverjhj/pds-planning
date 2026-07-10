# Session: 2026-07-10 — Item 6: Firebase staging/production split

**What changed:** Item 6 (the last pre-pilot build item) completed and glass-verified. Firebase is no longer a shared surface: a new **`pds-staging-xxxxx`** Firebase project (display name "PDS Staging"; Analytics and Gemini deliberately off — FCM-only, matching the app's firebase-messaging-only rule) now backs the staging environment, with the existing **`pds-production-xxxxxx`** untouched as production. The Android wiring turned out to be file placement only — the `staging`/`prod` flavours and the Google Services plugin already existed, and the plugin resolves per-flavour source sets natively — so the repo change is just a `.gitignore` widening (`app/src/*/google-services.json`) plus the CLAUDE.md docs update. Both flavour JSONs are placed locally (gitignored): `app/src/staging/google-services.json` and `app/src/prod/google-services.json`; the app-root copy is gone. The owner generated the staging service-account key (held in the local key store, outside all repos) and swapped staging's `FCM_SERVICE_ACCOUNT_JSON` Supabase secret to it — verified byte-exact by digest; production's secret digest confirmed unchanged (`fdd15059…`). No backend code change was needed: `audio-render-worker` derives the FCM project ID from the service-account JSON itself, so the secret swap re-pointed credentials and send URL atomically. The tablet was reinstalled with `stagingDebug` and self-healed its FCM token on launch, as predicted by recon (`RootViewModel` re-registers best-effort every launch).

**Commits:** pds-android `7daa273` — chore: split Firebase into per-environment projects (.gitignore + CLAUDE.md; reviewed and pushed by owner). pds-planning (this commit) — session log + STATE.md close-out.

**Decisions made:**
- **No app-root `google-services.json` fallback (fail-loud).** With a fallback present, a missing staging JSON would silently bind a staging build to the production Firebase project — the exact failure this item exists to kill. Per-flavour files only; a missing file fails the build loudly. Recorded in the Android CLAUDE.md (Environment configuration).
- **Secret-swap sequencing:** staging secret swapped before the tablet reinstall; the interim window is benign (per-device FCM failures are logged, never fail the render job; 30-min periodic sync is the fallback).
- **Ops, not architecture:** the Supabase MCP personal access token (both read-only servers) was found dead (401) mid-session and replaced by the owner with a fresh PAT — read-only posture and project pins unchanged (ledger #22 intact).

**Verified:**
- **Build-level:** `assembleStagingDebug` + `assembleProdDebug` both green; the plugin's generated `values.xml` checked per variant — staging → `pds-staging-xxxxx`, prod → `pds-production-xxxxxx` (prod task force-re-run fresh to rule out stale caching).
- **Glass (on the Lenovo tablet):** owner saved the Marford-to-Guildhall route on the staging dashboard → render worker run at 12:23:04 UTC (4.5 s, HTTP 200, no FCM errors) → **tablet synced hands-free seconds later**. Only the FCM push triggers that fast (periodic is 30 min), so the full staging chain — staging Firebase project → staging service account → staging-minted token — is proven end-to-end. Route `audio_render_status = ok`, fresh `audio_version` stamped.
- **Production:** secret digest unchanged after the swap; prod Firebase project, JSON, and function config never touched.

**What's next:** The pilot with the bus company (Stage 4's remaining substance), driven by owner-led iterative testing. No numbered build items remain.

**Banked / open:** (carried forward, plus session notes)
- Dashboard `/routes/[id]/edit` briefly 404'd in local dev — transient Turbopack cache artefact; a file touch forced the segment recompile and it resolved. No product bug; diagnostic log added and stripped in-session, dashboard tree clean.
- Supabase CLI on this machine is 2.98.2; 2.109.1 available — trivial, update whenever.
- Carried unchanged: ANDROID-3/4 Sentry issues (worth a look before the pilot); Android CLAUDE.md logcat `-s PDS` docs nit (dotted tags); pre-existing lint error `scripts/render-fixed-announcements.ts:172`; render-worker unchecked returned-`error` gap; three stale dashboard text sites; journey-summary cross-check for the 2026-07-10 GPS run.
