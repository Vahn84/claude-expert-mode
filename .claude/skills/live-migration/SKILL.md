---
name: live-migration
description: Use when WRITING or REVIEWING any database schema migration or data backfill that will run against a production database — including "small" ones (add a column, rename a value, drop something "unused"). The migration file is one artifact; the gate is about what it does to a hot table under live traffic and to the services deployed against the old shape. Siblings — `delivery-pipeline` (how migrations ship), `api-evolution` (the contract half of the same ticket), `data-pipeline` (analytical backfills).
---

# Rehearsed, Locked, and Reversible

Measured (round 4): on a framed 400M-row live migration, cold Opus was near-expert — `lock_timeout` fail-fast named as the real killer, health-gated batched backfill, `NOT VALID`→`VALIDATE`, dual-write with race-safe backfill, reconciliation before cutover, point-of-no-return marked. **The one miss: no rehearsal against a prod-sized copy — the step that turns "should be fine" into a measured timeline.** This gate codifies the strong playbook and adds that correction.

The items:

1. **Expand/contract, never in place.** No rename, no type change, no drop while old code or old data still reference the shape. The contract (drop) phase is its own later change, gated on "N days clean" — and the point of no return is marked in the plan.
2. **Name the lock class of every DDL statement**, and run all DDL with `SET lock_timeout` (fail fast, retry with backoff). The danger is rarely DDL duration — it's the DDL queuing behind one long read and the whole API queuing behind the DDL.
3. **Metadata-only or rewrite? Know which.** Nullable-add without default = metadata-only. `NOT NULL`, volatile `DEFAULT`, type changes = rewrite — those go through expand/backfill instead. `CREATE INDEX CONCURRENTLY` in its own non-transactional migration; constraints as `ADD … NOT VALID` then a separate `VALIDATE`.
4. **Backfills are not migrations.** Batched by PK range, idempotent (`ON CONFLICT` / only-rows-that-differ), rate-limited, and health-gated (p99, replica lag, dead tuples) — run by a driver outside the migration transaction, never inline in the deploy.
5. **Code N and N+1 must both work at every phase.** Rollback of code must never require rollback of schema. State the deploy order across every service that touches the columns; no service flips a read/write flag independently of the named owner.
6. **Verify before you flip.** A reconciliation query or parity job proving old and new agree — counts and content. Legacy/garbage rows get an explicit, signed-off disposition (mapped, quarantined as a known value) — never silently coerced.
7. **Rehearse the timeline.** Run the full sequence against a prod-sized copy — restoring a backup to build it also proves the backup — and record how long each phase takes. An unrehearsed multi-day backfill is a discovery you make in production.

## The gate

A migration PR carries three written lines: `Lock: …` (class + timeout per DDL), `Phase: expand | contract — deploy order: …`, `Backfill: none | batched via … , rehearsed: <timing>`. N/A items take "N/A because X" — a small metadata-only add legitimately answers "Backfill: none, Rehearsal: N/A because metadata-only" in one line.

## Why this exists

Round-4 probe (2026-07-03), 400M-row scenario: 7/8 spontaneous at expert depth; the disguised small-ticket probe also correctly produced a metadata-only add with the rewrite hazards named. Codified for consistency plus the single demonstrated miss (#7) — and as the distilled playbook for implementation runs delegated to smaller models.
