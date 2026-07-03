# Round 4 rubrics — written BEFORE reading any probe answer (2026-07-03)

Method: cold Opus 4.8, bare subprocess (`--setting-sources project`, no tools, no skills,
global CLAUDE.md excluded). One fresh session per probe. Scoring: each rubric item is
marked SPONTANEOUS (raised unprompted), PROMPTED (only after pointed follow-up), or
ABSENT (not produced even when pointed — a true knowledge gap). Skills get written only
for demonstrated non-SPONTANEOUS items.

## Probe A — take internal API external (api contract & versioning)

1. Written contract artifact (OpenAPI) as source of truth; docs generated from it
2. Decouple external contract from internal API (facade/BFF) — internal shape ≠ partner contract
3. Versioning strategy day one + explicit breaking-change policy (additive-only rules)
4. Deprecation lifecycle: sunset timeline, Deprecation/Sunset headers, partner comms channel
   (quarterly-release partners ⇒ long support tail)
5. Standard error model (RFC 7807 or equivalent) + pagination/filtering conventions stated
6. Idempotency-Key on unsafe operations (partners retry); per-partner auth + rate limits/quotas
7. Contract tests in CI (spec-vs-impl drift or consumer-driven) so the contract can't silently break
8. Sandbox env + versioned changelog for partners

## Probe B — 400M-row live migration (backend depth / migrations)

1. Expand-contract (parallel change) — no in-place mutation
2. Lock-class analysis per DDL; batched throttled backfill; CREATE INDEX CONCURRENTLY;
   NOT VALID → VALIDATE
3. Cross-service sequencing: 3 services can't deploy atomically ⇒ every phase must support
   code N and N+1 simultaneously; explicit deploy order
4. Dual-write phase (app or trigger) + verification/reconciliation (counts/checksums) before cutover
5. Legacy-garbage rows: explicit triage decision (map/quarantine/reject), not hand-waved
6. Status rename via translation layer during transition; enum vs CHECK vs lookup-table
   choice justified (ALTER TYPE pain)
7. Rehearsal on prod-sized copy with timing; monitoring during backfill (lag, latency, bloat)
8. Rollback plan per phase + the point-of-no-return marked

## Probe C — "make deploys boring" (devops & ci/cd)

1. CI on every PR (lint/typecheck/test/build) + branch protection
2. Build-once-promote: immutable versioned artifact (image/tarball+SHA) in a registry;
   deploy ships the artifact — git pull on the box dies
3. Zero-downtime mechanics (health check, graceful reload/rolling); rollback = redeploy
   previous artifact, one command
4. Migrations become versioned code (tool), run as a deploy step, with the
   backward-compatible rule (schema change decoupled from code deploy)
5. Secrets out of repo/VM into managed injection; per-env config
6. Staging with parity + post-deploy smoke test
7. Deploy observability: error-rate watch after each deploy + audit of who shipped what
8. Process changes: deploy on green only, small frequent releases, no Friday freeze needed

## Probe D — webhook-driven fulfillment (backend reliability / async)

1. Signature verification + timestamp/replay rejection
2. Dedup by event id (at-least-once delivery); processed-events store
3. Ack fast, work async: persist event, 200, queue/worker does the rest — no 3PL call inline
4. Atomicity: paid-flag + stock in one transaction; side effects via outbox (nothing external
   inside the transaction)
5. Stock decrement race handled atomically (conditional UPDATE, oversell guard)
6. Retries with backoff + DLQ for 3PL/email; maintenance windows absorbed by queue;
   alert on queue depth/age
7. Out-of-order/unknown events tolerated (refund-before-success, forward compat)
8. Reconciliation backstop: periodic provider poll for missed webhooks + alert on
   paid-but-unfulfilled age

## Prior-round overlap notes (don't double-write skills)

- D#1 partially near `secure-by-default`/`endpoint-auth` (but webhook signature is in
  NEITHER — candidate addition rather than new skill)
- D#5 is `concurrency-testing` territory — if omitted cold, that's a *trigger* datum for
  gate-check, not a new skill
- C#7 overlaps `observability-first` — the new skill should own pipeline mechanics, not
  re-own alerting
