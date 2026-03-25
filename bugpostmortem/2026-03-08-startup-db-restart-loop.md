# 2026-03-08 Startup DB Restart Loop Incident

## Summary

On March 7-8, 2026, the `binance-quant-vault` service experienced an abnormal restart cycle during startup. The container was eventually healthy again, but it was not continuously healthy for the full period.

Based on the observed logs:

- `2026-03-07 23:21:12Z`: one Bun runtime crash appeared as `panic: Segmentation fault`
- `2026-03-07 23:23:34Z` to `2026-03-07 23:45:33Z`: repeated database startup failures appeared as `Database connectivity check failed` and `ECONNREFUSED`
- target database: `database-1.cybuoyueck2q.us-east-1.rds.amazonaws.com:5432`
- `2026-03-07 23:45:37Z`: `Database connection test successful`
- after recovery, the service resumed normal behavior with recurring `heartbeat`, `binanceDataDelayMonitor`, and `binanceFuturesCollection` logs

## Impact

- The service could not finish startup while the database was temporarily unreachable.
- `docker compose` restart policy kept bringing the container back, creating noisy restart churn instead of a stable degraded state.
- Startup status could be misread as "service issue" or "container issue" when the real blocker was upstream database connectivity.
- The old Bun runtime added extra risk because a runtime crash was observed during the same incident window.

## Root Cause

The immediate failure mode was startup coupling:

1. Application startup required an immediate successful database connectivity check.
2. If that first DB check failed, the process exited.
3. Docker restart policy (`unless-stopped`) then restarted the container.
4. If the database was still unavailable, the same failure repeated.

This created a crash loop during temporary database outages, even though the correct behavior should have been "stay alive, keep retrying, become ready when the database comes back".

There was also a separate runtime-risk signal:

- the production image still used `oven/bun:1.1.20`
- the incident window included one `Segmentation fault`
- while the DB outage explains the restart loop, the outdated Bun version increased operational risk and made diagnosis noisier

## Why This Was Easy To Miss

- Once the database recovered, the service returned to normal and logs looked healthy again.
- Simple `docker ps` or tailing only the latest logs could hide the earlier restart storm.
- A running container is not equivalent to a healthy startup path.

## Implemented Fix

The codebase was hardened in four places.

### 1. Startup DB retry instead of fail-fast exit

`DbService` now retries startup connectivity checks with bounded backoff:

- `1s`
- `2s`
- `5s`
- `10s`
- `30s`
- then stays at `30s`

Each retry logs:

- attempt number
- database host and port
- retry delay
- error summary

The scheduler still does not start business jobs until DB connectivity succeeds.

### 2. Independent container healthcheck

The image now includes a dedicated `dist/healthcheck.js` entrypoint that verifies:

- secrets can be loaded
- database connectivity works

This allows Docker health status to reflect real readiness instead of only process existence.

### 3. Deployment script waits for `healthy`

`start-service.sh` no longer prints success after a fixed short sleep.

It now:

- waits until the app container is both `running` and `healthy`
- times out after 10 minutes
- prints recent container logs on failure

This prevents false-positive deploy success messages.

### 4. Bun runtime upgrade

The Docker image was upgraded from Bun `1.1.20` to Bun `1.3.9` to align with the version used in local verification and reduce runtime crash risk.

## Prevention Rules

Do not repeat the old pattern for any new service or startup dependency.

### Startup rules

- Do not make the container exit immediately on the first transient network or DB failure.
- If an external dependency is required for readiness, retry in-process with backoff.
- Separate `process is alive` from `service is ready`.

### Healthcheck rules

- Every non-trivial service must expose a real readiness check.
- Healthchecks must verify critical dependencies, not just the main process PID.
- Deployment scripts must wait for `healthy`, not just `running`.

### Dependency timeout rules

- Every DB connection path must have:
  - connection timeout
  - query timeout
  - statement timeout
- Every external HTTP call must have an explicit timeout.
- Every scheduled job that depends on network or DB must have an execution timeout.

### Runtime rules

- Keep local verification runtime and container runtime aligned.
- Avoid running stale Bun versions in production once the repo is validated on a newer version.
- Treat unexpected runtime crashes as infrastructure signals, not "probably harmless" noise.

## Operational Checklist

Use this checklist before and after every production deploy.

### Before deploy

- Confirm the image Bun version matches the version used in local validation.
- Confirm `docker-compose.yml` includes a real healthcheck.
- Confirm startup scripts wait for `healthy`.
- Confirm DB and external API timeout settings are still present.

### During deploy

- Watch for repeated startup logs mentioning:
  - `Database connectivity check failed`
  - `ECONNREFUSED`
  - `Segmentation fault`
  - `unhealthy`
- Do not treat the deploy as successful until health status becomes `healthy`.

### After deploy

- Confirm logs show:
  - `Database connection test successful`
  - `应用程序已成功启动`
  - recurring business job logs, not only `heartbeat`
- Confirm `docker compose ps` shows the service as healthy.

## What To Check First If This Happens Again

1. Database reachability to the configured host and port.
2. Container health status, not only process status.
3. Whether startup retries are happening as expected.
4. Whether the image version drifted from the repo's validated Bun version.
5. Whether the service eventually recovers after DB restoration without manual restart.

## Key Lesson

A temporary upstream outage should degrade readiness, not crash the service into a restart storm.
