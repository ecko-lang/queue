# queue

A durable job queue with leases, written in Ecko over `std.sql`: enqueue work,
a worker claims it with a lease, and a worker that dies (or hangs) has its job
reclaimed once the lease expires, until the job runs out of attempts and is
dead-lettered. No capabilities of its own: every function takes a database
handle you opened.

## Install

```bash
ecko get github.com/ecko-lang/queue
```

```ecko
import queue
```

## Usage

```ecko
import std.sql
import queue

db = sql.open(":memory:")  # sql.open("jobs.db") to share one queue between workers
queue.init(db)

queue.enqueue(db, "emails", { to: "a@example.com" })
queue.enqueue(db, "emails", { to: "bad" }, max_attempts: 1)

fn send_email(p) {
    if p.to == "bad" { error({ kind: "smtp", message: "no such mailbox" }) }
    print("sent to {p.to}")
}

# A worker pass: claim until nothing is ready.
mut job = queue.claim(db, "emails", "worker-1", 30000)  # 30 second lease
while job != null {
    try {
        send_email(job.payload)
        queue.complete(db, job)
    } catch (e) {
        print(queue.fail(db, job, get(e, "message")).status)
    }
    job = queue.claim(db, "emails", "worker-1", 30000)
}
# sent to a@example.com
# dead

s = queue.stats(db, "emails")
print("{s.done} done, {s.dead} dead")  # 1 done, 1 dead
print(queue.dead(db, "emails")[0].last_error)  # no such mailbox
```

### Enqueue with your own writes

Pass the `tx` of your own transaction and the job commits with your rows, or
not at all:

```ecko
import std.sql
import queue

db = sql.open(":memory:")
queue.init(db)
sql.exec(db, "create table orders (id integer primary key)")

sql.transaction(db, fn(tx) {
    sql.exec(tx, "insert into orders (id) values (1)")
    queue.enqueue(tx, "receipts", { order: 1 })
})
print(queue.stats(db, "receipts").ready)  # 1
```

### Retrying dead jobs

```ecko
import std.sql
import queue

db = sql.open(":memory:")
queue.init(db)
queue.enqueue(db, "emails", { to: "x@example.com" }, max_attempts: 1)
job = queue.claim(db, "emails", "w", 30000)
queue.fail(db, job, "smtp down")

for j in queue.dead(db, "emails") {
    queue.retry_dead(db, j.id)
}
print(queue.stats(db, "emails").ready)  # 1
```

## API

Every function takes the `db` handle from `sql.open` (or a `tx` inside
`sql.transaction`) as its first argument. Every function that reads the clock
takes `now:` (Unix ms), defaulting to `time.now()` - pass it in tests.

| function | returns |
|---|---|
| `init(db)` | `null`, after creating the queue's table and claim index if they do not exist |
| `enqueue(db, queue_name, payload, delay_ms: 0, run_at: null, max_attempts: 5, now: null)` | the new job's id |
| `claim(db, queue_name, worker_id, lease_ms, now: null)` | the oldest ready job under a fresh lease, or `null` |
| `complete(db, job, now: null)` | `null`, after marking a claimed job done |
| `fail(db, job, reason, backoff_ms: 1000, backoff_max_ms: 60000, now: null)` | `{ status, attempts, run_at }`, after scheduling a retry or dead-lettering the job |
| `extend(db, job, lease_ms, now: null)` | the job with a refreshed `lease_expires` |
| `stats(db, queue_name, now: null)` | `{ ready, scheduled, in_progress, expired, done, dead, total }` |
| `dead(db, queue_name, limit: 100)` | the dead-lettered jobs, newest first |
| `retry_dead(db, id, now: null)` | `null`, after moving a dead job back to ready with its attempts reset |

**`enqueue`:** `run_at` (an absolute Unix-ms time) or `delay_ms` (relative to
`now`) says when the job becomes claimable - `run_at` wins if both are given.
`max_attempts` is how many times the job may be claimed: with the default 5
it runs at most five times, and the fifth `fail` (or a fifth lease that
expires) dead-letters it. `max_attempts: 1` dead-letters on the first failure.

**`claim` returns** `{ id, queue_name, payload, attempts, max_attempts,
worker_id, lease_token, lease_expires }`, or `null` when nothing is ready.
`payload` is the value passed to `enqueue`. Pass the whole map to `complete` /
`fail` / `extend` - they use `id` and `lease_token` together to check you
still hold the job.

**`fail`:** `backoff_ms` is the delay after the first failed attempt; it
doubles on each one after, capped at `backoff_max_ms`. `status` is `"ready"`
or `"dead"`.

## Notes

### Errors

| kind | reason | when |
|---|---|---|
| `queue` | `lease_lost` | the job is not `"in_progress"` under this `lease_token` any more - already finished, or reclaimed by another worker |
| `queue` | `not_found` | the job's row is gone (`complete`/`fail`/`extend`), or `retry_dead` was given an id that names no dead job; the error carries `id` |
| `queue` | `not_storable` | `enqueue` was given a payload holding a decimal, record, secret, bytes or function |
| `bug` | `bad_argument` | an empty queue name or worker id, a count or duration that is not a positive (or non-negative) Int, a non-Int `now` or `run_at`, something that is not a claimed job, or an options map |
| `sql` | | the database refused a read or write; a brief "database is locked" during `claim` is retried first |

```ecko norun
import std.sql
import queue

db = sql.open("jobs.db")
job = queue.claim(db, "emails", "worker-1", 30000)
try {
    queue.complete(db, job)
} catch (e) {
    match get(e, "reason") {
        "lease_lost" => print("job {job.id} was reclaimed - discarding this result")
        _ => error(e)
    }
}
```

### Payloads

A payload is stored as JSON, so `enqueue` takes only what JSON gives back
unchanged: null, bools, ints, floats, strings, and lists and maps of those.
A decimal, record, secret, bytes or function anywhere inside it is refused
with `queue`/`not_storable` rather than claimed later as something else
(a decimal as a float, a secret as the text `"[secret]"`). Convert first -
`string(amount)` for a decimal, a map for a record.

### Leases

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

### Concurrency

**Workers.** Run each worker on its own connection: `sql.open` the same
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
A job only ever needs to be reclaimed when a worker asks for one, and
`claim` covers that case: it reclaims an expired lease that has attempts to
spare, and dead-letters one that expired on its last attempt, so a job that
kills its worker every time stops after `max_attempts` instead of looping.

**Job rows are kept, not deleted.** `done` and `dead` rows stay in the table
so `stats` and `dead` can report on them. If you run a very high volume
through one table for a long time, periodically deleting old `done` rows
yourself (`sql.exec(db, "delete from ecko_queue_jobs where status = 'done' and updated_at < ?", [cutoff])`)
keeps it from growing without bound; this package does not do that for you,
since "how long to keep history" is a decision only the caller can make.

## Testing

```bash
ecko test
```

Offline and deterministic - every test uses a fresh in-memory database and an
explicit `now`, apart from one that runs four workers on a temporary file.

## License

[MIT](LICENSE)
