# Session: 2026-07-12 — Admin PIN 4-digit relax + email deliverability audit

**What changed:** Two pieces of work, one shipped and one investigative.

1. **Admin PIN minimum relaxed 6 → 4 digits (FR-AT-48).** Owner request — the driver admin menu is not a brute-force target. Single behavioural change: `AdminPinHasher.MIN_PIN_LENGTH` 6 → 4, which flows to both the `isValidPinFormat` submit gate and the `requireValidPin` defensive check. Everything else was keeping the surface consistent: the on-box label `admin_set_pin_hint` "New PIN (6+ digits)" → "(4+ digits)" (the one the owner still saw reading "6+" on glass after the first pass — it only shows *before* submit, so the first edit which touched the post-submit `admin_feedback_invalid_format` string had missed it), the invalid-format string, and five doc-comment sites (`AdminPinHasher`, `AdminPinBackoff`, `AdminPinStore`, `AdminPinViewModel`, `AdminSurface`). Tests: the two `sub-6-digit rejected` cases → sub-4 (3-digit input), plus a new `four-digit pin is accepted` test, plus updated `isValidPinFormat` assertions. Existing PINs are unaffected (stored salt/hash/iterations unchanged; 6-digit PINs still satisfy the new floor). The pairing-code "6-digit" strings (a separate feature) were deliberately left untouched.

2. **Email deliverability audit (read-only, nothing changed).** Prompted by an owner question about auth emails (signup confirmation / password reset) landing in junk. Queried live public DNS for `pds-dashboard.com` against 8.8.8.8. **Finding: the authentication foundation is genuinely solid and passes DMARC** — this is a correct, standard Resend setup, so junk-foldering is NOT an auth failure. Live records:
   - **DKIM** — valid RSA key at `resend._domainkey.pds-dashboard.com`, `d=pds-dashboard.com` → aligns **strict** with `From: noreply@pds-dashboard.com`. (Key is 1024-bit — passes; 2048 is the modern preference.)
   - **SPF** — `send.pds-dashboard.com` = `v=spf1 include:amazonses.com ~all`; the Resend/SES return-path lives on this subdomain → aligns **relaxed** with the root From.
   - **Bounce MX** — `send.pds-dashboard.com MX 10 feedback-smtp.eu-west-1.amazonses.com` (EU, consistent with the EU stack) → Resend processes bounces/complaints, protecting reputation.
   - **DMARC** — `_dmarc.pds-dashboard.com` = `v=DMARC1; p=none;` — valid but **bare** (no `rua=`, so zero reporting visibility).
   - **Apex** — no SPF and no MX at the root itself (not needed for the current flow since the return-path is on `send.`, but a restrictive apex SPF would be anti-spoofing hardening).

   By elimination, the actual causes of junk-foldering are **new-domain reputation** (`pds-dashboard.com` is days old, zero sending history) and the **sparse Supabase default email template** (one line + one link, HTML-only — reads like phishing to filters). Recorded as a single deferred optimisation item in STATE.md's Defer Without Guilt.

**Commits:**
- `pds-android` `d6f056f` — Relax admin PIN minimum from 6 to 4 digits (7 files; pushed to origin/main, human-approved).

**Decisions made:**
- **Admin PIN minimum is 4, not the frozen-spec 6** — a deliberate, owner-requested divergence from the v3.9 FR-AT-48 "6+" spec, justified under "docs are evidence, not authority." The security tradeoff (keyspace 10^6 → 10^4) is accepted: the driver admin menu is not a brute-force target, and the durable retry backoff (`AdminPinBackoff`) + Keystore-bound encrypted storage remain the real defences. Not promoted to a numbered ledger decision (minor); the rationale lives here and in the code comments.
- **The email DNS setup is confirmed correct — do not "fix" the authentication.** The deferred work is reporting/content/reputation, not auth records.

**Verified:** Admin PIN — owner glass-verified on the Lenovo tablet that a 4-digit PIN now sets and unlocks. Unit tests updated to match (not run this session — offered). Email audit — read-only DNS queries against Google's public resolver; no send-side test (mail-tester / Postmaster) run yet.

**What's next:** No forced next step — owner-led iterative testing continues. When the email item is picked up, the highest-value first moves are DMARC `rua=` reporting + a mail-tester.com score to measure, then the template enrichment.

**Banked / open:**
- Email deliverability optimisation is now a Defer-Without-Guilt item (non-urgent; worth doing before the pilot operator is signing up, since a verification email in their junk is a bad first impression).
- The substantive remaining build item is unchanged: the **payment gate** (What's Left #7). Plus the pilot and the pre-pilot prod-test-data cleanup.
