# Findings — pg-boss inside a Supabase Edge Function (Deno)

## Verdict: **B — works, with caveats**

pg-boss 10.1.6 runs cleanly inside a short-lived Supabase Edge Function isolate
via Deno's `npm:` bridge. The primitives the plan depends on (`start`,
`createQueue`, `fetch`, `complete`, `fail`) all work, the retry state machine
survives across isolates (it's DB-driven, not timer-driven), and
`boss.maintain()` can be invoked manually from a worker tick. The caveats:
internal maintenance and scheduled jobs are *not* run automatically because we
must opt out of supervision, so archive/purge has to be driven by us; and the
Edge Function wall-clock budget caps how long pg-boss work can run per tick.

---

## Setup

- Local Supabase: CLI 2.98.2, runtime image
  `supabase-edge-runtime-1.73.13` (Deno 2.1.4), Postgres 15.
- Worker function: `supabase/functions/audio-render-worker-spike/index.ts`
  imports `npm:pg-boss@10.1.6`, instantiates with
  `{ schedule: false, supervise: false }`, awaits `start`, then
  `createQueue` → `fetch` (one job) → `complete` *or* `fail` → `stop`.
- Enqueue from host: `enqueue.mjs` (Node 24, `pg-boss@10.1.6` via npm,
  pinned with `--save-exact`).
- DB URL inside the function is auto-injected by the runtime as
  `postgresql://postgres:***@db:5432/postgres` — no manual wiring needed.

The function returns a structured log of every step with timings, so each
invocation is self-documenting. Raw responses are in `logs/`.

## What worked

1. **`npm:pg-boss@10.1.6` import.** No Deno/Node incompatibilities. The
   library's internal `pg` driver, advisory locks, and prepared statements
   all execute fine in the Deno isolate.
2. **`boss.start()` from a fresh isolate.** First-ever call (with schema
   bootstrap) was ~50 ms; subsequent invocations on the same isolate were
   8–14 ms. The schema migration is idempotent and only does work on first
   call.
3. **`boss.createQueue()`.** Idempotent in v10; ~1–10 ms.
4. **`boss.fetch('test-queue')`.** Returns an array (possibly empty) and
   completes in 2–5 ms. Crucially, it **does not block** when the queue is
   empty — it returns `[]` and the tick exits cleanly. Tested explicitly
   via `?mode=empty-ok`.
5. **`boss.complete(queue, id, output)`.** Marks the job `completed` in
   ~3 ms. The five-job drain test ran clean: all five jobs ended in state
   `completed`.
6. **`boss.fail(queue, id, { reason })`.** Marks the job `retry` (if under
   `retry_limit`) or `failed` (if exceeded). Tested the full lifecycle:
   - Enqueue 1 job with `{ fail: true }` payload.
   - Tick 1: `boss.fetch` returns the job, worker calls `boss.fail` →
     state = `retry`, `retry_count = 0`, `retry_limit = 2`.
   - Tick 2: same job re-fetched, fails again → `retry_count = 1`.
   - Tick 3: fails again, exceeds limit → state = `failed`,
     `retry_count = 2`.
   This **proves retries do not require a long-lived consumer**.
7. **`boss.maintain()` from a short-lived isolate.** Returned in ~5 ms,
   no errors. The instance also exposes `expire`, `archive`. (No `purge`
   in v10's public surface.)
8. **Wall time per tick.** Including network: 90–110 ms. Internal pg-boss
   work: 15–30 ms. Comfortably below any plausible Edge Function CPU
   budget.

## What didn't work / concerning behaviour

1. **Supervision must be disabled.** pg-boss's `supervise: true` (default)
   starts an in-process loop that runs maintenance every 60 s, plus
   monitoring intervals. In a short-lived isolate that's pointless at best
   and harmful at worst — those timers fire after the request handler has
   returned, racing the isolate's wall-clock termination. I disabled
   `supervise` and `schedule` in the constructor and that was enough to
   stop the timers from arming. **Anyone building this for real must set
   `{ supervise: false, schedule: false }`** or they'll see leaked timers
   and possibly partial work that gets killed by the runtime mid-statement.
2. **`schedule: false` means `boss.schedule()` is dead.** Cron-style
   scheduled jobs (`boss.schedule('every-monday', '0 0 * * 1', ...)`) rely
   on the supervisor. The plan doesn't use these (the worker *is* the
   scheduled tick), but it's a sharp edge if requirements expand.
3. **Isolate lifetime is not unbounded.** The functions runtime logged
   `wall clock duration warning` and `early termination has been
   triggered: isolate: …` after ~60 s of accumulated wall time across
   reused invocations. This is normal for Edge Functions and not a
   pg-boss problem, but it means: do not assume any state held in the
   isolate (cached `boss` instances, JS-side counters) persists. Treat
   every tick as cold. Currently we already do — `new PgBoss(...)` per
   request — and the cost is acceptable (8–14 ms warm, ~50 ms first-ever).
4. **`pgboss.job` is partitioned by queue.** v10 creates one partition
   table per queue (e.g.
   `pgboss.j5c945284fdfaf41de2c10209ebe65d128ac30b9032b0ff393809697c` for
   `test-queue`). This is a strong architectural opinion: every new queue
   = a DDL operation = a row in `pgboss.queue` + a new partition. If the
   plan ever creates queues dynamically per tenant / per stop / per route
   that would be a problem. For a small fixed set (e.g. `audio-render`,
   `audio-render-retry`) it's fine.

## Answers to the specific questions in the brief

### Q1. Can `boss.fetch` / `boss.complete` be used reliably from a short-lived scheduled Edge Function?

**Yes.** Both are pure DB operations and behave deterministically. `fetch`
returns immediately on empty queue, doesn't hang. `complete` is a simple
UPDATE. Across 8 invocations covering all four modes (default, fail, maint,
empty), the only non-instant operations were `boss.start()` cold (49 ms
first time, schema migration runs there) and warm (8–14 ms). All
operations completed before isolate termination.

### Q2. Does the retry mechanism work correctly across invocations, or does it require a long-lived process?

**It works across invocations and does *not* need a long-lived process.**
Retry state lives in `pgboss.job.state` + `retry_count`, transitioned by
`boss.fail()`. The next `boss.fetch()` from a *different* isolate picks
the job up again as soon as `start_after` is past. Confirmed with a
3-tick poison-job test: 0 → 1 → 2 → `failed`. The only timing element
is `retry_delay` (set on enqueue or via queue config) and `start_after`
— both stored on the row, so the next short-lived isolate that fetches
will honour them.

### Q3. Does pg-boss's internal maintenance (archive / expiry) need a separate mechanism?

**Yes** — but the mechanism is cheap: call `boss.maintain()` on the
worker tick (or on a separate, lower-frequency tick). With
`supervise: false` pg-boss does *not* run its internal maintenance loop,
which means `archive`, `expire`, and dead-letter routing stop happening
unless we drive them. `boss.maintain()` from a short-lived isolate
completed in ~5 ms in this test — calling it once every N ticks (say,
hourly) from the worker would cost essentially nothing. The plan needs
one extra line: an "is this tick a maintenance tick?" branch.

### Q4. If we ditched pg-boss and hand-rolled a queue table with FOR UPDATE SKIP LOCKED, what would we lose?

What we'd lose:

- **Schema and migrations.** pg-boss handles its own schema versioning
  (the `pgboss.version` table), partitioning, indexes, and upgrades.
  Rolling our own means we own that DDL forever.
- **Retry state machine.** `created` → `active` → `retry` → `failed` is
  baked in, with `retry_count`/`retry_limit`/`retry_delay`/`retry_backoff`
  fields and the transitions encapsulated in `boss.fail()`. Reproducing
  that as raw SQL is straightforward but error-prone (especially
  exponential backoff).
- **Dead letter routing.** v10 has first-class `dead_letter` column +
  config. Hand-rolling means another column and another bit of routing
  logic.
- **Job expiry / visibility timeout.** `expire_in` + the active-job
  watchdog (normally run by the supervisor, but you can run it via
  `maintain()`) returns crashed jobs to `retry`. Without it, a worker
  that crashes mid-job leaves the job stuck in `active` forever.
- **Singleton / unique jobs.** `singleton_key` + `singleton_on` give you
  "only one of this kind of job at a time" for free.

What we'd gain:

- **Smaller blast radius.** A single table with explicit columns is
  easier to reason about, debug, and migrate than pg-boss's seven-table
  schema with per-queue partitions.
- **No "supervise: false" footgun.** Hand-rolled means no timer story
  at all — every behaviour is explicit DB state.
- **No "queue = partition" constraint.** Hand-rolled queues can multiplex
  thousands of "logical queues" through a single table column.

## Recommendation

**Keep pg-boss, with explicit adjustments to the v3.8 plan:**

1. **Pin the version**: `npm:pg-boss@10.1.6` in the function;
   `pg-boss@10.1.6` (exact, via `--save-exact` or `~`-free) in
   `package.json` for the enqueue side. Re-pin deliberately on upgrade.
2. **Mandate `{ supervise: false, schedule: false }`** in the
   constructor. Document this as a hard requirement in the plan's
   "worker shape" section. The default settings are wrong for this
   runtime.
3. **Make maintenance an explicit responsibility**: every Nth tick (e.g.
   once an hour), the worker also calls `boss.maintain()`. Decide N
   and write it down. Cheap insurance against active-job leaks if a
   worker isolate is terminated mid-job.
4. **Plan around fixed queue names.** The partition-per-queue design
   means dynamic queue creation is a DDL hot path. If the design has
   per-route or per-tenant queues, revisit; otherwise fine.
5. **Document the budget.** Worker tick CPU budget needs to cover
   `boss.start` (warm ~15 ms) + `boss.fetch` (~5 ms) + the actual audio
   work + `boss.complete` (~3 ms) + `boss.stop`. The audio rendering
   itself is the dominant cost, not pg-boss.

If any of those adjustments are unacceptable, replace with a hand-rolled
`FOR UPDATE SKIP LOCKED` queue — it's a one-day build and the things
you'd be giving up (above) are all things you can rebuild as needed
later.

## Reproducibility

All raw outputs in `logs/` (numbered 01–10), the worker function in
`supabase/functions/audio-render-worker-spike/index.ts`, the enqueue script
in `enqueue.mjs`. To reproduce from a clean checkout:

```
supabase start
npm install pg-boss@10.1.6 --save-exact
supabase functions serve --no-verify-jwt &
node enqueue.mjs 5
curl -X POST 'http://127.0.0.1:54321/functions/v1/audio-render-worker-spike' \
  -H 'Authorization: Bearer <publishable key>'
supabase db query "select state::text, count(*) from pgboss.job group by 1"
```
