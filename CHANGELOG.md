# Changelog

## 0.56.0

### Breaking

- Every function takes its options as named arguments instead of an options
  map. Migration: `queue.enqueue(db, "q", p, { delay_ms: 60000, max_attempts: 3 })`
  becomes `queue.enqueue(db, "q", p, delay_ms: 60000, max_attempts: 3)`, and
  likewise `claim(..., now: t)`, `complete(db, job, now: t)`,
  `fail(db, job, reason, backoff_ms: 500)`, `extend(..., now: t)`,
  `stats(db, q, now: t)` and `dead(db, q, limit: 10)`. Passing the old map is
  refused with a message showing the new form.
- `enqueue` refuses a payload that JSON would not give back unchanged (a
  decimal, record, secret, bytes or function, anywhere inside it) with
  `queue`/`not_storable`. A decimal used to be claimed as a float, a secret as
  the text `"[secret]"`. Migration: convert before enqueueing
  (`string(amount)`, a record to a map), and keep secrets out of jobs.
- A job whose lease expires on its last allowed attempt is dead-lettered
  instead of being claimed again. Migration: none needed for a worker that
  calls `fail`; a job that crashes its worker now stops after `max_attempts`
  claims and shows up in `dead` with a `last_error` saying the lease expired.

### Added

- `retry_dead(db, id)` moves a dead job back to ready with its attempts reset.

### Fixed

- `enqueue` can run inside your own `sql.transaction`: pass it the `tx` and the
  job commits or rolls back with your other writes. It used to fail with "a
  transaction is already open on this db handle".
- A job that kept killing its worker was reclaimed forever, its attempts
  climbing past `max_attempts`, and never reached `dead`.
- Options are validated where they are passed: a negative or zero `lease_ms`,
  `max_attempts: 0`, a negative `delay_ms`, a non-Int `now` or `run_at`, an
  empty queue name or worker id, and a value that is not a claimed job are
  refused with `bug`/`bad_argument` naming the function and the option.
- The README said `max_attempts` is how many `fail` calls a job survives; it
  is how many times the job may run, so `max_attempts: 1` dead-letters on the
  first failure.
