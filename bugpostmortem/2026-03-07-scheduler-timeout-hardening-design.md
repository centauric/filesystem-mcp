# Scheduler Timeout Hardening Design

**Date:** 2026-03-07

**Problem**

On March 3, 2026, the `binanceFuturesCollection` and `binanceDataDelayMonitor` jobs stopped producing follow-up runs while the process continued to emit `heartbeat` logs. The scheduler stayed alive, but individual job fibers could block indefinitely on external dependencies. Because the scheduler loop waits for each job execution to finish before scheduling the next run for that job, a single hung execution permanently stalled future runs for that job.

**Goals**

- Ensure a hung scheduled job fails within a bounded time and releases its scheduler lock.
- Ensure Binance HTTP requests cannot block forever.
- Ensure database queries used by scheduled jobs have bounded execution time.
- Preserve the current retry model: after timeout or failure, the job retries on the next interval.
- Produce clearer logs and an incident record for future debugging.

**Non-Goals**

- Replacing the scheduler with a cron or fixed-rate engine.
- Adding distributed locking.
- Reworking all services to use a new timeout framework.

## Approaches Considered

### 1. Scheduler-only timeout

Add a timeout around each job execution in `SchedulerService`.

Pros:
- Smallest code change.
- Prevents permanent scheduler lock retention.

Cons:
- Root cause remains opaque.
- HTTP and DB work can still hang internally until the outer timeout fires.

### 2. Multi-layer timeout hardening

Add a scheduler-level timeout, HTTP request timeout, and DB query timeout.

Pros:
- Fixes the production failure mode directly.
- Produces clearer failure boundaries.
- Allows each layer to fail with more useful error messages.

Cons:
- Slightly larger change set.
- Requires a few focused tests.

### 3. Fixed-rate scheduler redesign

Change the scheduler from `sleep -> run -> sleep` to independent fixed-rate tickers.

Pros:
- More standard scheduling semantics.

Cons:
- Larger behavioral change than needed.
- Higher regression risk.

**Recommendation:** Approach 2. It addresses the actual failure mode without unnecessary scheduler redesign.

## Design

### Scheduler

- Extend job definitions with an optional `timeoutMs`.
- Apply a default scheduler timeout to every job unless overridden.
- When a timeout occurs:
  - fail the current run,
  - log a timeout-specific error,
  - release the running lock in `ensuring`,
  - continue to the next interval.

This preserves the current scheduler model while making it resilient to hung effects.

### Binance HTTP layer

- Wrap Binance `fetch` calls with `AbortController`.
- Abort requests after a bounded timeout.
- Map aborts to a dedicated `HttpError` message so logs clearly show timeout vs. HTTP status errors.

### Database layer

- Add `query_timeout` to the pg pool configuration so client-side waits are bounded.
- Add `statement_timeout` via connection options so PostgreSQL server-side execution is also bounded.
- Keep connection timeout as-is for initial connection establishment.

### Logging

- Keep existing job start and finish logs.
- Add explicit timeout messages for scheduler-level timeouts.
- Preserve existing failure logging so operators can grep for `failed`, `timeout`, and job names.

### Tests

- Scheduler test: a hanging job times out and runs again on the next interval.
- Binance service test: a stalled `fetch` rejects with a timeout-specific error.
- DB config test: pool config includes bounded query and statement timeouts.

## Incident Notes

The March 3 incident showed two separate failure modes:

- database connectivity degraded first for `binanceDataDelayMonitor`,
- at least one later job run hung long enough to prevent future scheduling for that job.

The scheduler did not crash, so `heartbeat` kept printing and masked the fact that the data jobs were no longer making progress.
