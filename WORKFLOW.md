# WORKFLOW.md
# Passenger Display System (PDS) — Operating Manual

**Version:** 3.7
**Last Updated:** May 2026

## Changelog

### v3.7 (May 2026)
Post-adversarial-review re-planning pass, item 7 of 7 (compliance, WORKFLOW, smaller items, and campaign close-out).
- §1.2: Clarified two-repo reality — Android app repo and dashboard repo each have their own CLAUDE.md; builder reads only the repo it is working in.
- §5.2: Added compliance cross-check ownership — architect produces acceptance checklist; system administrator verifies on tablet against that checklist.
- §9 intro: Fixed singular "repo" to reflect two repos.
- §9.1: Replaced stale "upload phase" canonical CLAUDE.md example with "stops sync as a unit with their parent route."
- Quick Reference: Replaced single CLAUDE.md row with two rows (Android repo, dashboard repo).
- §10.3 (new): Room migration policy — every Room schema change requires a Migration object or an explicit fallbackToDestructiveMigration with rationale; must be tested on a real device with pre-existing data before commit.

---

This document defines how we work on this project. It is the operating manual that codifies the architect/builder split, the three-window setup, the plan-mode-first ritual, commit cadence, and other conventions hard-won from the previous attempt.

**Frozen-documents principle.** Four documents are frozen at the end of planning and are not edited during the build: PRD.md, Data-Architecture.md, Compliance-Mapping-Matrix.md, and this document. They represent the contract for what we're building and how. If during the build the architect believes one of these documents needs to change, that signals a planning failure, not a routine amendment. The correct response is to stop the build, return to planning, revise the documents deliberately, then resume. This rule exists to prevent the scope-drift pathology that ruined the previous attempt. See section 13 for details.

Evolving project information — decisions made during the build, discoveries, course-corrections — lives in CONTEXT.md, not in the frozen documents.

---

## 1. The Three Surfaces

The project operates across three distinct surfaces. Each has a defined role, and the separation matters.

### 1.1 Architect — Claude Project (this Anthropic project)

The architect is Claude operating inside the dedicated Anthropic project workspace. Its role is **strategic planning, design decisions, and prompt authoring**. The architect does not write production code.

The architect has access to the following project knowledge files:
- `PRD.md` — what the product is
- `Data-Architecture.md` — how data is shaped
- `Compliance-Mapping-Matrix.md` — legal compliance evidence
- `WORKFLOW.md` — this document
- `CONTEXT.md` — living memory of project state, decisions, and session history

The architect reads these to design tasks. It does not need to read the codebase directly because CONTEXT.md tells it what exists.

The architect's outputs are:
- Prompts for claude-code (the builder)
- Reviews of claude-code's plans
- Updates to CONTEXT.md (the only knowledge file edited during the build)
- Architectural decisions captured in CONTEXT.md's decisions log

### 1.2 Builder — Claude Code in VS Code

The builder is Claude Code running inside VS Code, with file system access to the repository. Its role is **implementation**: writing, editing, and refactoring code; running builds and tests; committing to git.

The builder reads:
- `CLAUDE.md` in the repo it is working in. **There are two repositories and two CLAUDE.md files:** the Android app repo has its own CLAUDE.md covering Kotlin/Hilt/Room/GPS conventions; the dashboard repo has its own CLAUDE.md covering Next.js/TypeScript/Supabase conventions. The builder reads only the CLAUDE.md for the repo it is working in during that session. It never reads the other repo's CLAUDE.md.
- The actual code in the repo
- The prompt the architect provides

The builder does *not* read the PRD, Data-Architecture, Compliance Matrix, or WORKFLOW. Those documents are for the architect. If the builder needs a rule from one of those documents, the architect distils it into CLAUDE.md or into the specific prompt.

### 1.3 Tester — Android Studio + Lenovo Tablet

Build verification happens on a real device (Lenovo tablet, connected via USB) using Android Studio. An emulator is acceptable for UI-only work, but anything involving GPS, foreground services, audio routing, or kiosk mode is tested on the physical tablet.

The system administrator (you) operates Android Studio manually. The builder can request builds via `./gradlew` but cannot deploy to the tablet — that's a human action via Android Studio.

### 1.4 Why the Separation

The previous attempt collapsed when the architect (Claude in chat) lost sync with the builder (Claude-code in VS Code). The architect was making decisions based on an imagined codebase; the builder was making decisions based on documents the architect hadn't seen. The two drifted until the codebase was unsalvageable.

The separation re-establishes clear roles. Architect designs. Builder builds. Tester verifies on real hardware. CONTEXT.md is the architect's view of reality; CLAUDE.md is the builder's view of conventions; the running tablet is the ultimate source of truth.

---

## 2. The Plan-Mode-First Ritual

Every claude-code task follows this ritual. No exceptions.

### 2.1 The Flow

1. **Architect writes prompt.** Based on CONTEXT.md state, PRD requirements, and current task scope, the architect writes a prompt for claude-code. The prompt includes: what to build, which spec or PRD requirement it implements, any specific constraints or conventions.

2. **System admin pastes prompt into claude-code.** No editing of the prompt unless trivial typos. If the prompt seems wrong, send it back to the architect.

3. **Claude-code enters plan mode.** Claude-code reads relevant files and produces a plan: what it will change, in what order, with what approach.

4. **System admin pastes plan back to architect.** Always. Even for routine tasks. This is the project's primary defence against drift.

5. **Architect reviews plan.** Returns one of:
   - "Looks good, proceed."
   - "Looks good with these clarifications: [...]"
   - "Plan needs changes: [...]"

6. **System admin acts on architect's response:**
   - If "looks good, proceed": click **"Yes, and auto-accept edits"** in the plan dialog.
   - If "looks good with clarifications": click **"Yes, and auto-accept edits"**, then paste the architect's clarifications as a follow-up message in claude-code.
   - If "plan needs changes": click **"No, keep planning"**, paste the architect's required changes as the next message, claude-code revises, restart from step 4.

7. **Claude-code executes.** With auto-accept on, claude-code makes edits without per-file prompts.

8. **System admin verifies on tablet** (see section 5).

9. **System admin commits** (see section 4).

10. **System admin reports outcome to architect.** Usually one or two sentences: "Built and tested successfully" or "Hit issue X — see below." The architect updates CONTEXT.md if state changed.

### 2.2 The Critical Pitfall

If the architect's plan review contains *any* corrections, do NOT click "Yes, and auto-accept" then paste corrections. Click "No, keep planning" and revise the plan first. Auto-accepting a flawed plan means claude-code is halfway through the wrong implementation by the time corrections arrive.

Clarifications are different from corrections. A clarification is "also handle empty state, you didn't mention it" — claude-code can absorb this mid-flight. A correction is "you're using the wrong table" — claude-code needs the plan rewritten before executing.

If in doubt, treat it as a correction and revise the plan.

### 2.3 Why Always Send the Plan Back

Two reasons:

1. **Reality check.** The architect designs prompts based on CONTEXT.md, which is the architect's mental model. Claude-code reads the actual code. If the plan reveals that the actual code state differs from what CONTEXT.md says, the architect knows immediately and can update.

2. **Catching scope creep.** Claude-code sometimes proposes more than was asked ("while I'm here, I'll also refactor X"). The architect catches this.

The overhead is small — a 30-second read per plan — and the protection is large.

---

## 3. Effort Levels

Claude-code supports four effort levels: **low, medium, high, max**. Default is high in the UI, but for this project we use a different mental model.

| Task type | Level |
|---|---|
| Trivial fixes (typo, single-line bug fix, renaming a variable) | **low** |
| Routine implementation (build a screen from spec, add a DAO method, write a use case) | **medium** |
| Multi-file architectural work (new feature spanning ViewModel + repository + DAO + entity, refactoring across modules) | **high** |
| Hard correctness problems (sync algorithm, GPS state machine, race conditions, anything where "looks right" isn't enough) | **max** |

Default is **medium**, not high. High and max are escalations. Using high for routine work tends to produce over-engineered solutions: claude-code "thinks harder" and invents complexity that wasn't asked for. The previous build had this pathology; we deliberately avoid it.

The architect specifies the effort level in the prompt: "Effort: medium." or "Use max effort — this is sync correctness." If unspecified, system admin uses medium.

---

## 4. Commit Cadence

### 4.1 When to Commit

Commit after each working feature, where "feature" means one prompt that ends with a verified passing state on the tablet.

The unit is **prompt-and-verify**, not file-by-file. If a single feature requires three prompts to claude-code, that's still one commit at the end (assuming the intermediate states were either non-functional or working-but-incomplete).

If a feature is large enough that committing only at the end risks losing work, the architect breaks it into smaller prompts each with its own commit. This is the architect's responsibility when designing tasks.

### 4.2 Commit Messages

The architect provides the commit message at the end of each task in the format:

```
<type>: <short title referencing the PRD requirement>

<detailed description in 2-4 paragraphs covering:
 - what was changed and why
 - which PRD requirements this implements
 - any notable decisions or trade-offs>
```

`<type>` is one of: `feat`, `fix`, `refactor`, `docs`, `chore`, `test`.

Example:

```
feat: implement FR-AT-19 stylised tube-map route progress view

Adds a custom Android View (TubeMapView) rendering the route as a
horizontal line of stop nodes with the current position as a moving dot.
The view is the bottom third of the journey screen and re-renders only on
stop progression events plus a 30-second tick, per FR-AT-20.

Implements FR-AT-19, FR-AT-20, FR-AT-21. Compliance-relevant: the next-stop
node uses the high-contrast highlight scheme from FR-AT-37; the view does
NOT contain regulated text (text remains in the top section), so 22mm rules
do not apply here.

Trade-off: stops are evenly spaced rather than geographically accurate.
Justified because geographic accuracy would require continuous GPS-driven
re-rendering, which contradicts FR-AT-20's battery conservation goal.
```

### 4.3 Branching

Single `main` branch. Frequent commits. No feature branches — there's only one developer.

This implies pushing directly to `main`. Acceptable because the only consumer of `main` is the system administrator's own tablet for testing. If the team grows, this changes.

### 4.4 What Not to Commit

- `local.properties` (gitignored — contains Supabase keys)
- `.idea/` (Android Studio config)
- `*.apk` build outputs
- Anything that fails to build (verify build passes before committing)

---

## 5. Empirical Test Ritual

Every claude-code prompt ends with a verified state on the tablet. The level of verification depends on the work.

### 5.1 Standard Verification (most features)

For non-compliance features:
- Build passes (`./gradlew assembleDebug` or via Android Studio)
- App installs on the tablet without errors
- The implemented feature works end-to-end for the happy-path test the architect specified

The architect specifies the happy-path test in the prompt. Example: "Verify by tapping a route in the route list and confirming the route detail screen shows all stops in order."

### 5.2 Strict Verification (compliance-relevant features)

For anything compliance-relevant:
- All of the standard verification, plus
- The happy-path test
- At least one failure case (GPS lost mid-journey, Bluetooth disconnects, sync fails, etc.)
- Cross-check against the Compliance-Mapping-Matrix to confirm the relevant clause is satisfied

Compliance-relevant features include:
- Any audio announcement (next-stop, route, termination, diversion, hail-and-ride)
- Alert chimes
- 22mm text rendering and calibration
- High-contrast display
- GPS-based stop detection
- Audio output handling (Bluetooth, fallback)
- Mixed-case enforcement (no all-caps)
- Audio-visual consistency

The architect specifies "strict verification" in the prompt when this applies.

**Compliance cross-check ownership:** The "cross-check against the Compliance-Mapping-Matrix" is performed as follows. The architect, when writing the prompt for a compliance-relevant feature, produces a **compliance acceptance checklist** — a short list of the specific regulatory clauses the feature must satisfy, extracted from the Compliance Matrix and distilled into concrete pass/fail checks (e.g., "alert chime plays before termination audio on first try," "22mm text confirmed with ruler," "driver panel open: next-stop name still visible above panel"). This checklist is included in the prompt to claude-code. The system administrator runs these checks during tablet verification (step 8 of the plan-mode-first ritual). The builder (claude-code) does not read the Compliance Matrix and does not perform the cross-check — it reads the architect's distilled checklist from the prompt. The system administrator is the verifier; the architect is the checklist author.

### 5.3 What "Tested" Does Not Mean

It does NOT mean:
- "Compiles without errors" (necessary but not sufficient)
- "Passes unit tests" (unit tests can be wrong; tablet is ground truth)
- "Looks right in the code review"
- "Claude-code says it works"

The previous build failed in part because we declared things "done" without running them on the tablet. The schema-mismatch crash was caught only when we finally ran the app.

### 5.4 Failure Handling

If verification fails:
1. System admin reports the failure to the architect with as much specificity as possible (logcat output, screenshots, what was tried).
2. Architect diagnoses and produces a follow-up prompt for claude-code.
3. Cycle repeats from step 2 in section 2.

Do NOT commit a failed state. The repo's `main` should always be working (per section 4.4).

---

## 6. When to Start a New Claude-Code Chat

### 6.1 Rule

**New claude-code chat per feature.**

A "feature" is a unit of work corresponding to one or more closely-related PRD requirements. Examples:
- "Build the first-run pairing flow" (one feature, even though it spans setup screen + Edge Function call + JWT storage)
- "Add the route list screen" (one feature)
- "Implement GPS-based stop detection with sequential progression" (one feature)

Multi-task features stay in one chat. The chat ends when the feature is committed and tested.

### 6.2 Mid-Feature Chat Reset

If claude-code's context gets muddled mid-feature (it forgets what it just did, references files it didn't create, repeats earlier mistakes), start a fresh chat even mid-feature. Pass the relevant context via the architect's next prompt.

### 6.3 What Goes in a New Chat's First Prompt

The architect's prompt for the start of a new chat includes:
- What's being built (feature name + PRD reference)
- Effort level
- Verification expectations (standard or strict, with happy-path scenario)
- Any specific files claude-code should read first (e.g., "read CLAUDE.md and `app/src/main/java/com/pds/application/data/local/dao/RouteDao.kt`")
- Whether plan mode is required (default: yes, always)

Claude-code reads CLAUDE.md automatically at chat start, so we don't need to repeat that instruction every time.

---

## 7. The Architect Chat (This Project)

### 7.1 One Long Chat Is Fine

Unlike claude-code chats, architect chats can stay long. This Anthropic project chat has held lengthy planning sessions without degradation. The architect benefits from continuity within a chat — designing PRD, then Data-Architecture, then WORKFLOW all in one session keeps the documents internally consistent.

### 7.2 When to Start a New Architect Chat

Start a new architect chat when:
- The current chat hits the compression notice (Anthropic compresses long chats; once compressed, fidelity drops)
- The current chat feels "muddled" — the architect is contradicting itself or losing earlier context
- A major phase ends and a fresh slate would help (e.g., "we've finished planning, now starting build")

### 7.3 What to Do When Starting a New Architect Chat

1. Update CONTEXT.md to reflect current state.
2. Re-upload all knowledge files to the project (or confirm they're up to date).
3. Open a new chat in this project.
4. The architect reads CONTEXT.md and the knowledge files automatically — no re-priming needed.

Optionally, paste in a one-line "what we're working on right now" to give the new chat immediate orientation. Example: "We've finished drafting CLAUDE.md; next task is starting Stage 1, the web dashboard MVP."

---

## 8. CONTEXT.md Update Ritual

CONTEXT.md is the architect's living memory. It is the **only** document that changes during the build phase, and it is the single document that re-uploads between architect chats. Everything that evolves about the project — decisions, discoveries, course-corrections, current state — flows through CONTEXT.md. The frozen documents (PRD, Data-Architecture, Compliance, WORKFLOW) do not move during the build.

### 8.1 When CONTEXT.md Gets Updated

Updates happen when the architect or the system admin notices the current chat has been productive enough that the state-of-the-world has materially changed and would matter to a future architect chat. There is no fixed cadence.

Triggers include:
- Completion of a planning document (e.g., PRD finalised → log decision in CONTEXT.md)
- Completion of a feature in the codebase (e.g., route list screen built and tested → reflect in current-state)
- An architectural decision worth preserving (e.g., "use Supabase Auth uniformly for both surfaces")
- Approaching a context compression
- Starting a new chat

The architect typically prompts: "Worth updating CONTEXT.md now?" The system admin says yes or no.

### 8.2 Structure of CONTEXT.md

Three sections, always:

**Current state** — what exists in the codebase right now. Brief inventory: which features are built, which are in progress, which are planned. Updated when state changes.

**Decisions log** — dated entries of architectural decisions. Each entry: date, what was decided, what alternatives were considered, why this. Append-only — don't delete past decisions, even if they're later overridden (the override is a new entry).

**Session history** — brief notes per architect-chat session. Each entry: date, what got done in that session, what's next. Append-only.

### 8.3 How CONTEXT.md Gets Updated

The architect produces an updated CONTEXT.md (either via the `create_file` tool or by stating exactly what to add and the system admin doing it manually). The system admin saves the new version and re-uploads to the Anthropic project knowledge.

Subsequent architect chats see the new version immediately.

---

## 9. CLAUDE.md Maintenance

CLAUDE.md lives in the root of each repository. There are two: one in the Android app repo and one in the dashboard repo. Each is read by claude-code at the start of a session in its respective repo.

### 9.1 When to Update CLAUDE.md

Add to CLAUDE.md when:
- A new convention is established that future tasks should follow (e.g., "all repository interfaces require operator_id scoping")
- A gotcha is discovered that future claude-code should avoid (e.g., "when adding a new Room entity, also update PdsDatabase.entities[]")
- A new architectural rule is documented (e.g., "route stops sync as a unit with their parent route — stops are never independently created, modified, or synced")

Do NOT add to CLAUDE.md:
- Feature implementation details (those go in code comments or commit messages)
- One-off decisions specific to a particular feature
- Anything that would only matter for one claude-code session

The test is: would this rule matter for at least three different future tasks? If yes, it goes in CLAUDE.md. If no, leave it out.

### 9.2 Who Edits CLAUDE.md

The architect proposes changes to CLAUDE.md. Claude-code does not edit CLAUDE.md autonomously — if claude-code wants to add a rule, it suggests the edit and the architect decides.

This rule exists because in the previous build, the builder gradually rewrote its own conventions, drifting from the original intent.

---

## 10. Safety and Caution

### 10.1 Things Claude-Code Should Not Do Without Explicit Instruction

- Run database migrations or schema-modifying SQL against Supabase (architect must explicitly authorise; see 10.4 on the Supabase MCP read-only default)
- Delete files outside the current task's scope
- Modify CLAUDE.md (see 9.2)
- Push to GitHub (use local commits only; pushes are manual — note that the GitHub MCP is connected but used only for read-only repo/issue/PR operations unless the architect specifically authorises a write)
- Install dependencies that change major versions of core libraries (Hilt, Room, Kotlin, Next.js) without architect approval

These are listed here so the architect can reference them in CLAUDE.md and in prompts.

### 10.2 Things to Watch For

- Claude-code "discovering" files that contradict CONTEXT.md → reality has drifted from the architect's model, update CONTEXT.md immediately
- Plans that propose more than the prompt asked → scope creep, push back
- "While I'm here, I'll also..." patterns → say no, keep the task scoped
- Plans that don't reference the PRD requirement they're implementing → architect probably forgot to include it in the prompt, fix prompt and restart

### 10.3 Room Migration Policy (Android App)

The previous build's fatal failure was a Room schema mismatch — the app crashed on launch because the local database schema did not match the expected version. This policy prevents that class of failure.

**Rule:** Any change to the Android Room schema (adding/removing/renaming a table, column, or index) must be accompanied by one of:
- A Room `Migration` object implementing the schema change for existing databases, **OR**
- An explicit `fallbackToDestructiveMigration()` call with a written rationale comment in code (acceptable only during early development when no production data exists on any real device).

**Testing requirement:** Room migrations must be tested on a real device with **pre-existing data** before committing. The test procedure: install the old APK, create representative local data (sync a route, start a journey, so Room has populated rows), then install the new APK over it and verify the app launches correctly and existing data is intact. This cannot be replaced by unit tests or emulator-only testing.

**When to invoke this policy:** Every time a Room entity class, DAO, or database class is modified. The architect must confirm the migration approach in the prompt for any task that touches Room schema. Claude-code must flag to the system admin if a schema change is being made and no migration approach was specified in the prompt.

**Rationale:** The previous build declared the schema "done" without testing on a device with existing data. The mismatch was only discovered at runtime. One migration or one explicit destructive-migration decision, tested once on a real device, prevents the crash.

### 10.4 MCP Server Configuration and Safety

Claude-code is configured with three MCP servers at the user scope: **Context7** (live library documentation), **GitHub** (read access to repos, issues, PRs), and **Supabase** (database and Edge Function operations). A fourth, **Playwright** (browser automation for dashboard verification), is to be installed before Stage 1 dashboard verification work begins — not yet installed.

The MCP servers expand what claude-code can do without manual relay (running SQL against Supabase, fetching current library docs, querying repo state). The expanded capability also expands the surface for unwanted side effects, so the following rules apply.

**Supabase MCP read-only default.** The Supabase MCP is configured with `--read-only` at all times by default. Claude-code can read schema, query data, list Edge Functions, and inspect storage — but cannot execute migrations, modify data, or deploy Edge Functions. Write access is granted only when the architect explicitly authorises a specific task that requires it. When write access is needed:

1. The architect's prompt explicitly states that write access is required for this task and why.
2. The system administrator reconfigures the Supabase MCP without `--read-only` (commands documented in personal operational notes).
3. Claude-code performs the authorised write task.
4. The system administrator reverts the MCP to `--read-only` immediately after the task completes.

Read-only is the default state to which the configuration always returns. The MCP is never left in read-write mode between tasks. This is enforced operationally, not by the tool itself.

**GitHub MCP scope.** The GitHub MCP token is granted Contents, Issues, Pull requests, and Metadata permissions. Claude-code uses it for reading repo state, drafting issues, and reading PRs. Writes (commits, pushes, merging PRs) remain manual — claude-code does not push to GitHub regardless of MCP capability. If a future task requires automated PR creation, that's a deliberate architect decision, not a default behaviour.

**Context7 has no auth surface.** It fetches public library documentation. No special handling.

**Playwright (when installed) operates on local browsers only.** It runs against the dashboard's localhost dev server or a Vercel preview URL — not production. The architect must explicitly authorise running Playwright against any deployed environment.

**Token handling.** MCP server tokens (Supabase Personal Access Token, GitHub Personal Access Token) live in the user-level Claude Code config (`~/.claude.json` or platform equivalent). They are never committed to repos, never pasted into chats, and never shared in screenshots. If a token is exposed, revoke and regenerate before doing anything else.

---

## 11. The First Hour of a New Session

When starting a new architect-chat session, follow this checklist:

1. Confirm CONTEXT.md is current (re-upload if you've updated it since the last chat).
2. Open this chat (the architect project).
3. The architect reads CONTEXT.md and knowledge files automatically — no priming prompt needed unless you want to orient quickly.
4. If you're about to start a new feature, the first message can be: "Let's start [feature name]. Refer to PRD [requirement]."
5. The architect produces the first claude-code prompt.

When starting a new claude-code session:

1. Open a new Claude Code chat in VS Code.
2. Paste the prompt from the architect.
3. Wait for plan mode.
4. Paste the plan back to the architect.
5. Proceed per section 2.

---

## 12. Quick Reference

| Action | Where | Who |
|---|---|---|
| Design product features | Architect chat | Claude (architect) |
| Write code | Claude-code chat | Claude-code (builder) |
| Build & deploy | Android Studio + tablet | System admin |
| Plan review | Architect chat | Claude (architect) |
| Commit | Claude-code chat or terminal | Claude-code (with architect-provided message) |
| Push | PowerShell terminal | System admin |
| Update CLAUDE.md | Claude-code chat | Claude-code (architect-approved) |
| Update CONTEXT.md | Architect chat | Claude (architect) |
| Modify PRD / Data-Arch / Compliance / WORKFLOW | Not done during build — see section 13 | — |

| Document | Lives in | Read by |
|---|---|---|
| PRD.md | Claude Project knowledge | Architect |
| Data-Architecture.md | Claude Project knowledge | Architect |
| Compliance-Mapping-Matrix.md | Claude Project knowledge | Architect |
| WORKFLOW.md | Claude Project knowledge | Architect |
| CONTEXT.md | Claude Project knowledge | Architect |
| CLAUDE.md (Android) | Android repo root | Builder (claude-code) — Android tasks |
| CLAUDE.md (Dashboard) | Dashboard repo root | Builder (claude-code) — Dashboard tasks |

| MCP server | Installed | Purpose | Default |
|---|---|---|---|
| Context7 | Yes | Live library documentation lookup (Next.js, Supabase JS, Hilt, Room, etc.) | Always on |
| GitHub | Yes | Read repos, issues, PRs; drafting only; no pushes | Read-only by convention |
| Supabase | Yes | Schema/query/Edge Function access | `--read-only` flag; write only on explicit authorisation |
| Playwright | Not yet — install before Stage 1 dashboard verification | Browser automation for dashboard end-to-end checks | When installed, localhost/preview only |

| Effort level | Use for |
|---|---|
| low | Trivial fixes |
| medium | Routine implementation (default) |
| high | Multi-file architectural work |
| max | Hard correctness (sync, GPS, races) |

---

## 13. The Frozen Documents Rule

This section formalises the rule introduced at the top of this document. It is one of the most important rules in the project and warrants its own section.

### 13.1 Which Documents Are Frozen

Four documents are frozen at the end of the planning phase:

- `PRD.md`
- `Data-Architecture.md`
- `Compliance-Mapping-Matrix.md`
- `WORKFLOW.md` (this document)

Once planning ends and the build begins, these documents do not change.

### 13.2 Why

The previous attempt failed in part because the PRD was treated as a living document. The architect amended it as the build progressed: a new feature would be added when claude-code "discovered" it might be useful, scope would creep, the PRD would be retroactively edited to match what was built. By the end, the PRD described an imagined product that bore only partial resemblance to the codebase, and the codebase contained features that had never been deliberately planned.

Freezing the documents prevents this. The PRD describes what we agreed to build. The build delivers it. Discrepancies are caught early because they cannot be papered over by editing the spec.

### 13.3 What "Frozen" Means in Practice

- The architect does not edit the frozen documents during the build phase
- Claude-code does not read or edit the frozen documents in normal task work
- The system administrator does not edit the frozen documents in response to build-time discoveries
- The frozen documents are read-only artefacts in the Claude Project knowledge

### 13.4 What to Do When You Think a Frozen Document Needs to Change

Two cases, with different responses:

**Case 1: The frozen document is wrong.**
You believe the PRD says something the product should not do, or omits something the product must do, or the Data-Architecture specifies something that won't work.

Response: this is a planning failure. Stop the build at the next clean break (do not abandon mid-task work that's already started, but do not start new work). Open a new architect chat. Treat it as a re-planning session: discuss what's wrong, decide what to change, produce a revised frozen document, increment its version number, re-upload to the Claude Project. Then resume the build.

This should be rare. If it happens more than once or twice during the build, the planning was insufficient and we should pause for a more thorough re-plan.

**Case 2: The frozen document is fine; you need more detail.**
You believe the PRD adequately specifies what to build but claude-code needs additional information that wasn't in the document (e.g., "the PRD says 'tube map view' but doesn't specify the colour scheme").

Response: this is normal. The detail goes into the architect's prompt to claude-code, or into CLAUDE.md if it's a convention that'll recur. The frozen document is not edited.

The test for distinguishing the cases: if the missing detail is something that could reasonably have been left out of the PRD (because it's too granular for a product spec), case 2. If the missing detail is something the PRD should have addressed but didn't (because we genuinely failed to think about it), case 1.

### 13.5 The Frozen Documents and CONTEXT.md

CONTEXT.md captures everything that evolves during the build. The frozen documents capture what was decided during planning. Together they describe the project's full history:

- Frozen documents: "what we agreed to build, as of the start of the build"
- CONTEXT.md decisions log: "additional architectural choices made during the build, refining the frozen agreements"
- CONTEXT.md current state: "what's actually been built so far"
- CONTEXT.md session history: "what each work session accomplished"

A future contributor reading the project should be able to read the frozen documents first (for the contract), then CONTEXT.md (for everything since). The two together are the complete record.

### 13.6 Version Bumps

If the frozen documents *do* need to change (case 1 above), the new version increments the document's version number (e.g., PRD v3.0 → PRD v3.1). The old version is preserved in git history. The Claude Project knowledge is updated with the new version.

This happens at re-planning time, never mid-build. There is no concept of "the architect quietly updated PRD.md to match what we ended up building" — if the agreement changes, it changes deliberately, and the version number reflects that.

---

## End of WORKFLOW.md
