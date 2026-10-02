# queue - Ecko Std Lib Package

A durable job queue with leases, for [Ecko](https://ecko.sh), written in Ecko
over `std.sql`. Enqueue work, a worker claims it with a lease, and a worker
that dies (or hangs) has its job reclaimed once the lease expires. An
attempt/lease token stops a reclaimed job from being finished twice.

Meant for the kind of work where losing a job or running it twice costs
money: transaction processing, backfills, anything you would not want to
re-run by hand and cannot afford to silently drop.

## Install

```bash
ecko get github.com/ecko-lang/queue
```

This package asks for no capabilities of its own - it never opens a database
connection itself, only runs queries on the handle you give it, and `sql.exec`
/ `sql.query` / `sql.transaction` are not gated. Opening a file-backed
database is what needs `fs`, on whichever package or top-level code calls
`sql.open`.

## Usage

```ecko
import std.sql
import queue

db = sql.open("jobs.db")           # or ":memory:"
queue.init(db)                     # create the table once

queue.enqueue(db, "emails", { to: "a@example.com" })
queue.enqueue(db, "reports", { id: 9 }, { delay_ms: 60000, max_attempts: 3 })

job = queue.claim(db, "emails", "worker-1", 30000)   # 30 second lease
if job != null {
    try {
        send(job.payload)
        queue.complete(db, job)
    } catch (e) {
        queue.fail(db, job, string(e))
    }
}
```

A worker loop is just `claim` in a while-not-null, `complete` on success and
`fail` on error - see `example.ecko` for a full pass with retries and a
dead letter.

## API

Every function takes the `db` handle from `sql.open` as its first argument.
`opts` is always optional; every function that reads the clock takes
`opts.now` (unix ms) so it never has to touch the wall clock - pass it in
tests, leave it out in real code.

| Function | Description |
|---|---|
| `init(db)` | Create the queue's table and claim index if they don't exist yet. |
| `enqueue(db, queue_name, payload, opts?)` | Add a job. `payload` is JSON-encoded. Returns the job id. |
| `claim(db, queue_name, worker_id, lease_ms, opts?)` | Atomically claim the oldest ready job, or `null`. |
| `complete(db, job, opts?)` | Mark a claimed job done. |
| `fail(db, job, reason, opts?)` | Record a failed attempt: retries with backoff, or dead-letters past `max_attempts`. |
| `extend(db, job, lease_ms, opts?)` | Refresh a claimed job's lease. |
| `stats(db, queue_name, opts?)` | Counts: `{ ready, scheduled, in_progress, expired, done, dead, total }`. |
| `dead(db, queue_name, opts?)` | List dead-lettered jobs, newest first. |

**`enqueue` options:** `run_at` (an absolute unix-ms timestamp) or `delay_ms`
(relative to `now`, default 0) - `run_at` wins if both are given. Neither
delays claiming past when they say; `max_attempts` (default 5) is how many
`fail` calls the job survives before it is dead-lettered.

**`claim` returns** `{ id, queue_name, payload, attempts, max_attempts,
worker_id, lease_token, lease_expires }`, or `null` when nothing is ready.
`payload` comes back JSON-decoded. Pass the whole map to `complete` / `fail` /
`extend` - they use `id` and `lease_token` together to check you still hold
the job.

**`fail` options:** `backoff_ms` (default 1000) is the delay after the first
failed attempt; it doubles on each subsequent one, capped at
`backoff_max_ms` (default 60000). `fail` returns
`{ status: "ready" | "dead", attempts, run_at }`.

**`dead` options:** `limit` (default 100) caps how many rows come back.

## Notes

**Leases, not locks.** `claim` marks a job `"in_progress"` with a
`lease_expires` timestamp and a fresh, random `lease_token`. Another `claim`
call will not pick up that job until either it finishes (`complete`/`fail`)
or its lease expires - at which point `claim` treats it as ready again and
hands it to whoever asks next, with a new token. There is no separate
"reclaim" step or background sweep; every `claim` call already looks for
expired leases as part of the same query that looks for never-claimed ones.

**The lease token, not the clock, decides who owns a job.** If worker A's
lease expires but nobody has claimed the job yet, A can still `complete` or
`fail` it - `lease_expires` is only a hint for eligibility to reclaim.
Once worker B does reclaim it, B gets a new `lease_token`, and A's next
`complete`, `fail` or `extend` call raises instead of quietly succeeding or
colliding with B's work. This is what makes it safe for a slow worker to
finish a job late, and unsafe for a *dead* worker's late finish to corrupt
whatever the job's replacement has since done.

**Errors.** Operational failures from `std.sql` pass through unchanged (kind
`"sql"`). This package's own errors use `kind: "queue"` with a `reason`
field:

| `reason` | when |
|---|---|
| `not_found` | the job id no longer has a row (never happens in normal use; would mean the row was deleted out from under the queue) |
| `lease_lost` | the job isn't `"in_progress"` under this `lease_token` any more - already finished, or reclaimed by another worker |

```ecko
try {
    queue.complete(db, job)
} catch (e) {
    match get(e, "reason") {
        "lease_lost" => log.warn("job {job.id} finished twice or reclaimed - discarding this result")
        _ => error(e)
    }
}
```

**Concurrency.** Run each worker on its own connection: `sql.open` the same
database file once per worker, rather than sharing one handle across threads
or spawned tasks. `claim` is a single `update ... returning` statement, so
choosing a job and taking its lease happen under one write lock and two
workers can never claim the same job. When another worker holds that lock,
SQLite refuses at once with "database is locked"; `claim` treats that as
contention and retries briefly before giving up, so it reaches your code only
if the lock is held for well over a second. A `:memory:` database is private
to the connection that opened it, so it only supports a single worker; use a
file to run several workers against one queue.

**No background sweeper.** Nothing here polls for expired leases on its own.
That is deliberate - a job only ever needs to be reclaimed when a worker
asks for one, and `claim`'s query already covers that case, so there is
nothing for a sweeper to do that `claim` does not do lazily.

**Job rows are kept, not deleted.** `done` and `dead` rows stay in the table
so `stats` and `dead` can report on them. If you run a very high volume
through one table for a long time, periodically deleting old `done` rows
yourself (`sql.exec(db, "delete from ecko_queue_jobs where status = 'done' and updated_at < ?", [cutoff])`)
keeps it from growing without bound; this package does not do that for you,
since "how long to keep history" is a decision only the caller can make.

**Works across Ecko 0.58**, which changed `sleep` from seconds to milliseconds.
The short pause before retrying a locked database is 2 ms per attempt on either
side of that change; 0.55.0 passed a fraction to `sleep`, which 0.58 refuses.

## Testing

```bash
ecko test
```

Offline and deterministic - every test uses a fresh in-memory database and an
explicit `now`, so nothing touches the wall clock. Covers: enqueue/claim/
complete, claim returning `null` on an empty or fully-leased queue, two
claims never landing on the same job, lease expiry making a job reclaimable
and refusing the stale worker's `complete`, `fail`'s retry-then-dead-letter
path with backoff, delayed jobs staying unclaimable before `run_at`, and
`stats`/`dead`.

## License

MIT - see [LICENSE](LICENSE).
