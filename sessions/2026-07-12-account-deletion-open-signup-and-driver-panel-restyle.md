# Session: 2026-07-12 — Account deletion + open signup + driver-panel restyle

**What changed:** A pilot-facing feature + polish session (fifth same-day 2026-07-12 session). Discharged the two remaining owner-on-glass items from prior sessions, then shipped three things: (1) **open signup** — new operators become `active` immediately instead of `pending`; (2) a **self-service delete-account** feature on the dashboard that hard-wipes the operator and all their data; (3) **UI polish** — a dark-green-circle dashboard favicon and a full restyle of the tablet driver-control-panel buttons. A payment gate was recorded as the last remaining build item.

- **Open signup (dashboard).** `handle_new_user()` set `operators.status` via a hardcoded `'pending'` literal (migration 021). New migration **025** `CREATE OR REPLACE`s it to `'active'`. Deliberately minimal: only the initial value changed — the full three-state status machinery (middleware `/pending` + `/suspended` gates, RLS `operator_status = 'active'` mutation gate, enum CHECK) is retained, so `suspended` still works and re-adding approval later is a one-line migration. No existing rows migrated. The accepted "anyone can sign up" exposure is temporary — the planned **payment gate** becomes the real access control. Applied to staging AND prod (owner ran `supabase db push`); MCP-verified the trigger writes `'active'` on both.

- **Delete-account (dashboard).** New `delete-account` Edge Function (Rule 5 — service-role admin API): verifies the re-entered password (ephemeral anon client), enumerates device auth users, wipes the operator's `route-audio` Storage prefix, deletes device auth users, then deletes the operator's auth user LAST — which cascades away every Postgres row (`operators.user_id → auth.users ON DELETE CASCADE` + the operator-CASCADE fan-out). Front half: a `deleteAccount` server action + a destructive "Danger zone" `AlertDialog` on the account page, password-confirmed, that signs out + redirects to `/login` on success. Deployed to staging AND prod (owner ran `supabase functions deploy`, byte-identical digest both projects). Owner glass-tested end-to-end on prod (created + deleted `owner@example.com`); MCP-verified clean (0 matching auth users, 1 operator remaining = the admin, 0 orphans on both envs).

- **Favicon (dashboard).** Added `src/app/icon.svg` (solid `#0d2a1c` circle, the dark-mode Booking Hall ground), removed the stale binary `src/app/favicon.ico`. Convention-based (no `metadata.icons`). Owner glass-confirmed the tab.

- **Driver-panel buttons (tablet).** The panel's filled buttons were mis-using the semantic amber `warning` role as a generic accent (the brown/gold). Restyled through a new `DriverPanelButton` wrapper. First iteration (de-amber + dashed "stamp" borders) was glass-rejected — the dashes read as tacky and bled over the amber/crimson fills. **Final form: every button is a flat SOLID fill, no border** — ink fill + ground text for every control (the ground↔ink inversion, "opposite of the background"), with End Journey in `ephemera` crimson and Emergency-Mute-engaged in `warning` amber. Amber is now confined to engaged-mute + the GPS-lost / speaker-disconnect markers. No new palette roles (reuses `onBackground`/`background`/`ephemera`/`warning`); `PassengerPaletteTest` untouched. Android Rule 20 + all three design-system copies (skill, DESIGN-LANGUAGE.md, Claude Design `driver-panel` card) updated per ledger #27.

**Commits:**
- pds-dashboard `32b0276` — feat: new operators start active — open signup (migration 025)  *(pushed)*
- pds-dashboard `428b4c6` — feat: delete-account Edge Function — service-role hard wipe  *(pushed)*
- pds-dashboard `1c99507` — feat: delete-account server action + Danger-zone dialog  *(pushed)*
- pds-dashboard `4a2a429` — feat: dark-green circle favicon  *(pushed)*
- pds-android `2fac05a` — feat: de-amber driver panel + Booking Hall stamp buttons (Rule 20)  *(pushed; superseded by ce23f0f)*
- pds-android `ce23f0f` — refactor: driver panel — solid buttons, drop the dashed stamp border  *(committed; **owner review + push + Lenovo glass still pending**)*
- pds-planning `8097349` — docs: design language — driver-panel buttons are solid, no border

**Decisions made:**
- **Open signup keeps the status machinery; only the initial value flips.** Payment gate is the intended future access control (new What's-Left item), not a re-added approval step — but re-adding one is a one-line migration if ever needed.
- **Account deletion is a hard wipe, owner-decided** (delete = delete). The auth-user cascade does the Postgres teardown; device auth users + Storage are the only explicit cleanups; operator-auth-user delete goes LAST so a mid-way failure leaves a still-deletable operator, not orphans. Confirmation is **password re-entry** (not typed-name). No admin exception — self-service delete will wipe the owner's own admin account too (accepted). The delete-with-data path (device users + Storage wipe) was **not** exercised on glass — owner explicitly declined it as a to-do (confident it works); recorded as accepted, not banked.
- **Driver-panel buttons: solid fills, no border** (dashed "stamp" idiom tried and rejected on glass). Amber is never a generic button accent — reserved for warnings + the muted-state indicator (Rule 20). End Journey stays the one distinct action (crimson).

**Verified:** Dashboard — `typecheck`/`build`/`lint` green across all commits; open-signup + delete-account deployed and owner-glass-tested on prod, MCP-verified clean; favicon glass-confirmed. Android — `compileStagingDebugKotlin` + `testStagingDebugUnitTest` (+ `assembleStagingDebug` on the first iteration) green; owner glass-verified the button restyle direction. `ce23f0f` (final solid form) is owner-glass-approved by screenshot but the built APK push + on-Lenovo re-check is still owner-pending.

**What's next:** The pilot (unchanged), owner-led. Immediate: owner reviews + pushes pds-android `ce23f0f` and installs on the Lenovo.

**Banked / open:**
- **pds-android `ce23f0f` pending owner review + push + Lenovo glass** (both themes: every button a solid block, no dashes, mute-engaged clean amber, End Journey crimson).
- **Payment gate / subscription system** — the last substantive build item before real launch (see What's Left To Do). Supersedes the removed signup approval as the access control.
- Delete-account **with data** (routes + devices + audio) is unexercised on glass — owner-accepted, deliberately not a to-do.
- The two prior banked glass items are **discharged this session** (owner-confirmed): admin calibration nav `29cc3c3`, and the 22 mm ticker ruler check mid-scroll + Lockito Tier-A diversion pass.
