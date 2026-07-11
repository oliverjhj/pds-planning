# Session: 2026-07-12 — Custom domain + the production-never-served fix

**What changed:** Took the dashboard from "deploys green but 404s on its public URL" to genuinely live on a custom domain. Bought `pds-dashboard.com` through the Vercel registrar (one-bill, ~$1 over wholesale — owner's call, deliberate) and connected it to the `pds-dashboard` project. Then chased down why every production URL 404'd, in two layers:

1. **Alias gap (surface).** The current production deployment (`6a5120e`) was built ~9h before the domain was bought, so its alias set never included `pds-dashboard.com` — the Domains page showed "Valid Configuration" but the domain was attached to nothing. A redeploy re-ran alias assignment and attached it (confirmed via the deployment's `alias` array over the Vercel MCP).
2. **Root cause (the real one).** After the domain was attached, the deployment *still* served Vercel's raw `404: NOT_FOUND` on every route — because the project's **Framework Preset was set to "Other", not "Next.js"**. With "Other", Vercel ran `next build` (so builds always went green) but then served static files from `public`/root and never wired up the Next.js runtime or routing manifest. **Implication: the dashboard's production URL had never actually served the app since Stage 1** — it went unnoticed because all real testing was local (`npm run dev`), the generated `*.vercel.app` URLs sit behind Vercel SSO (Deployment Protection), and the clean `-lovat` production alias had silently 404'd the whole time. Setting Framework Preset → **Next.js** + redeploy fixed it. Site now loads and works at `pds-dashboard.com`.

No code was written this session — it was entirely Vercel/Supabase configuration. `siteUrl()` already reads an env var, so the remaining auth-email wiring needs config only, not a commit.

**Commits:** none in `pds-android`/`pds-dashboard` (config-only session). `pds-planning` <this close-out commit> — session record + STATE.md.

**Decisions made:**
- **Custom domain bought through Vercel, not a third-party registrar** — owner's reasoning accepted: one billing surface, business write-off, ~$1 price delta vs Cloudflare wholesale not worth the extra DNS/SSL setup. `pds-dashboard.com` is now the production URL.
- **Framework Preset must stay "Next.js"** on the Vercel project. "Other" is the failure mode that produces green-but-unserved builds. Recorded as an operational anchor, not a ledger rule (it's config hygiene, not an architectural constraint).
- **Password recovery framing:** operator/owner dashboard passwords are bcrypt-hashed in `auth.users` and unrecoverable by design — there is nothing to "look up." Access is regained by the reset flow or a Supabase-console password set, never by reading the DB.

**Verified:** `pds-dashboard.com` loads and works (owner confirmed on glass). Diagnosis steps were MCP-verified: the production deployment's `alias` array (custom domain missing pre-redeploy, present post-redeploy), `framework: null` on the project, and live 404s on fresh (uncached) fetches of `/` and `/login` before the Framework Preset fix. Deployment Protection observed as Standard-style: generated `*.vercel.app` URLs 302 to Vercel SSO; the custom production domain is public.

**What's next:** Owner-led pilot testing continues (unchanged). First, close the auth-email wiring below so operator password resets work on the live domain.

**Banked / open (from the domain switch — flagged this session, NOT yet confirmed done):**
- **`NEXT_PUBLIC_SITE_URL` env var** (Vercel → pds-dashboard → Production) = `https://pds-dashboard.com`, then redeploy. Without it, `siteUrl()` falls back to `NEXT_PUBLIC_VERCEL_URL` (the per-deployment, SSO-protected URL), so signup-confirmation / email-change / password-reset links point at the wrong, gated URL. **Status: unconfirmed.**
- **Supabase production Auth URL Configuration** (project `qopkjsihmdplmsoqrzji` → Authentication → URL Configuration): Site URL = `https://pds-dashboard.com`; add `https://pds-dashboard.com/auth/callback` to the Redirect URLs allow-list. Supabase rejects any redirect target not on the list, so the two auth flows break without this even if the env var is set. **Status: unconfirmed.**
- **End-to-end verify** the password-reset email on the live domain — this both proves the two items above and regains the owner's dashboard login (the prompt that started the password question).
- **Deployment Protection mode** — confirm it's "Standard Protection" (custom domain public, which matches observed behaviour) rather than "All Deployments", so nothing silently gates the public site later.
