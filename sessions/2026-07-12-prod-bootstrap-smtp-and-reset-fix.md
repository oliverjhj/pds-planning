# Session: 2026-07-12 — Production go-live: admin bootstrap, branded email, reset-flow fix, analytics

**What changed:** Took the now-public dashboard from "serves the app" to "operationally usable by a real operator." Four threads, all owner-driven config plus two small commits:

1. **Production was an empty project.** A login-failure question surfaced that prod (`qopkjsihmdplmsoqrzji`) had **0 operators and 0 auth users** — the owner's dashboard account had only ever existed on **staging** (`febcpudiorwnsbcphcng`), which is where local `npm run dev` points (`NEXT_PUBLIC_SUPABASE_URL`). No password "mismatch"; prod simply had no such user, and Supabase's anti-enumeration meant the reset email silently never sent. **Bootstrapped the prod admin account:** confirmed the custom-access-token hook was already enabled on prod (Auth → Hooks, `public.custom_access_token_hook`), signed up through the live domain, then manually flipped `operators.status` pending→active in the prod SQL editor (no in-app approval UI — the owner is the admin). Owner now has a working production login.

2. **Auth-email wiring completed** (the open item banked from the custom-domain session): set `NEXT_PUBLIC_SITE_URL = https://pds-dashboard.com` in Vercel (Production) + redeploy; set prod Supabase Auth **Site URL** to the domain and added `https://pds-dashboard.com/auth/callback` (plus `/**`) to the redirect allow-list. Verified end-to-end by the confirmation + reset emails landing on the live domain. **Deployment Protection** confirmed = **Standard** (custom domain public, `*.vercel.app` behind SSO).

3. **Fixed a latent bug: the password-reset form was unreachable.** The recovery email link (`/auth/callback?token_hash=…&type=recovery`) calls `verifyOtp()` — which **logs the user in** — then redirected to `/reset-password`. That page lives in the `(auth)` route group, whose layout does `if (user) redirect('/')`, so every reset link dropped the user straight into the dashboard without ever showing a set-password form. **Fix (`79ad7f5`):** recovery now redirects to `/account?reset=1` — a dashboard page (no auth-layout bounce), reachable even for pending/suspended operators (middleware exempts account paths), whose existing change-password form needs only a new password (same `updateUser({ password })` the old reset form used). The account page shows a one-time banner and scrolls to the password card on `?reset=1`. Removed the now-dead `(auth)/reset-password` page + form, the `updatePassword` action, `UpdatePasswordSchema`/`UpdatePasswordInput`, and the orphaned `/login?reset=success` banner.

4. **Custom SMTP → PDS-branded production email** (unblocks deliverability + rebrands the sender off "Supabase Auth"). Set up **Resend** (Google login; region Ireland/eu-west-1), verified `pds-dashboard.com` by adding the DKIM/SPF/DMARC records into **Vercel DNS** (manual, not auto-configure), created a `supabase-smtp` API key, and entered custom SMTP into **prod** Supabase Auth (`smtp.resend.com:465`, user `resend`, key as password; sender **PDS** `<noreply@pds-dashboard.com>`). Raised the email rate limit to 30/h. **Glass-verified:** a live password reset arrived branded as **PDS** and landed on the `/account` password section — proving the SMTP *and* the reset-flow fix in one shot.

5. **Vercel Web Analytics added** (`a31764e`): `@vercel/analytics` + `<Analytics />` in the root layout; owner enabled the project toggle; `/_vercel/insights/script.js` confirmed serving on prod. Cookieless, so no consent-banner burden on the regulated product.

**Commits:**
- pds-dashboard `79ad7f5` — fix: land password-reset users on the account page (FR-WD-05)  *(owner-reviewed + pushed)*
- pds-dashboard `a31764e` — feat: add Vercel Web Analytics  *(owner-authorized Claude to push directly, one-off)*

**Decisions made:**
- **Prod admin bootstrap is manual** — no in-dashboard operator-approval UI; the owner signs up and flips `status` to `active` in the SQL editor. Same path any pilot operator's approval will take (owner approves manually — matches the deferred `retry-admin-notification` posture).
- **Custom SMTP (Resend) on prod only; staging stays on default Supabase email** — the default's ~2–3/h cap never bites solo staging testing, and staging mail only ever goes to the owner. Simplicity over symmetry (owner's call).
- **Recovery lands on `/account`, not a dedicated reset page** — the `(auth)`-group reset-password page was dead-on-arrival (auth-layout bounce) and is removed rather than patched.
- **Speed Insights deliberately skipped** — ~$10/mo real-user Core Web Vitals monitoring is low value for a low-traffic internal operator tool; free Lighthouse/PageSpeed covers any ad-hoc perf check.
- **Email-template Booking Hall rebrand declined** — owner likes the default reset template; sender identity (the branded bit) is what mattered and is done via SMTP.
- **CI/CD kept as-is** — pure Vercel Git integration (push `main` → prod `pds-dashboard.com`; branches/PRs → SSO-gated `*.vercel.app` previews on staging Supabase, via Vercel per-environment env vars). No GitHub Actions gate; owner happy with the simplicity. Recommended-but-not-adopted: a branch → preview(staging) → merge-to-main(prod) flow now that it's public.

**Verified:** Prod admin login works (owner on glass). Confirmation + password-reset emails deliver on the live domain, branded **PDS**, reset link lands on the `/account` password section (glass). Reset fix + analytics: `typecheck` / `lint` / `build` all green; `/reset-password` gone from the build's route list; `/_vercel/insights/script.js` serving on prod (confirms both the deploy and the analytics toggle). No tablet/glass work — dashboard-only, non-compliance surfaces.

**What's next:** Owner-led pilot testing continues (unchanged).

**Banked / open (all non-blocking, deferred by choice):**
- **Staging SMTP** — only if email testing on staging/local previews is ever wanted; prod is fully done.
- **Email-template Booking Hall rebrand** — declined for now; the one piece Claude can edit (Supabase → Emails → Templates) if the owner ever wants branded bodies.
- **Speed Insights** — skipped; enable later if dashboard perf ever matters.
- **GitHub Actions CI gate** + the branch→preview→prod flow — optional hardening, not pursued (owner prefers current simplicity).
