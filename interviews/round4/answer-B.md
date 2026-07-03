Here's the plan. Two independent expand/contract (parallel-change) cycles, run **sequentially** — `status` first (it's a single in-place column, good for proving the batching harness against the hot table), then `address` (structurally bigger but gentler on `orders` since the backfill only reads it). Never both DDL streams at once.

## Phase 0 — Guardrails (before any migration commit)

These apply to *every* step below; they're the difference between a clean rollout and a p99 incident.

- **Every DDL statement** runs with `SET lock_timeout = '2s'; SET statement_timeout = '...'`, and the migration runner retries on timeout with backoff. On 400M rows behind a 24/7 API, the killer isn't the DDL duration — it's a DDL queuing behind one long read and then *everything* queuing behind the DDL. `lock_timeout` makes us fail fast and retry instead of freezing the API.
- **Every index** is `CREATE INDEX CONCURRENTLY`.
- **Every constraint** is `ADD ... NOT VALID` then a separate `VALIDATE CONSTRAINT` (validate takes `SHARE UPDATE EXCLUSIVE` — reads/writes continue).
- **Every backfill** is batched by primary-key range, idempotent, rate-limited, and **health-gated**: the driver pauses if p99 rises above threshold, replica lag exceeds a bound, or dead-tuple count / autovacuum falls behind. Batches ~5–10k rows, sleep between.
- One named owner holds the flag-flip runbook; services never flip their own source-of-truth flag independently.
- Confirm PITR/backups and that you can restore to a timestamp. Bump autovacuum aggressiveness on `orders` for the duration.
- Build the observability first: dashboards for p99, replica lag, `pg_stat_activity`/`pg_locks` (lock waits), `n_dead_tup`, and batch progress.

---

## Track A — `status` free-text → constrained set (+3 renames)

**Design decision:** use a **`CHECK` constraint**, not a native `enum`. Enums can rename values but can't *remove* them without recreating the type, and you need non-canonical values to exist transiently while garbage is cleaned. A CHECK constraint is flexible and cheaply revertible.

**A0 — Discovery.** `SELECT status, count(*) FROM orders GROUP BY 1`. Enumerate the 11 known values, the 3 renames, and every garbage variant. Check whether any index exists on `status` (affects HOT-update / bloat behavior).

**A1 — Product decision, in writing.** Define the canonical target set (post-rename). Decide the garbage disposition per value: repaired to a real status where derivable, otherwise a quarantine value like `legacy_unknown` that is *in* the allowed set. Get sign-off — this is a data-semantics decision, not an engineering one.

**A2 — Ship the canonicalization layer.** A shared module used by all three services:
- **Writes** emit only canonical (post-rename) values.
- **Reads** translate old→new on the way out, so consumers see one consistent vocabulary while the table is mixed.
Deploy to all three services. No DB change yet.

**A3 — Add the constraint as `NOT VALID`.** Fast, metadata-only; enforces only *new* writes:
```sql
ALTER TABLE orders
  ADD CONSTRAINT orders_status_chk
  CHECK (status IN ('...canonical values...')) NOT VALID;
```

**A4 — Backfill, batched.** Only touch rows that actually differ (avoids needless bloat/WAL):
```sql
UPDATE orders
   SET status = <mapping(status)>
 WHERE id >= $lo AND id < $hi
   AND status NOT IN ('...canonical values...');
```
Loop over PK ranges, health-gated. `VACUUM (ANALYZE)` `orders` afterward.

**A5 — Validate.** Once backfill is complete and `SELECT count(*) ... WHERE status NOT IN (...)` returns 0:
```sql
ALTER TABLE orders VALIDATE CONSTRAINT orders_status_chk;
```

**A6 — Remove the read-time translation** from all three services (data is now uniformly canonical). Track A done.

**Rollback:** before A5, drop the constraint and the mapping is reversible; the canonicalization layer tolerates both vocabularies, so a bad deploy degrades to "some rows unmigrated," not an outage.

---

## Track B — JSONB `address` → normalized `addresses` table

**B1 — Create the table (no lock on `orders`).**
```sql
CREATE TABLE addresses (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_id    bigint NOT NULL,
  kind        text   NOT NULL,           -- 'billing' | 'shipping'
  line1 text, line2 text, city text, region text,
  postal_code text, country text,
  raw         jsonb,                      -- keep original for audit/repair
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT addresses_kind_chk CHECK (kind IN ('billing','shipping'))
);
CREATE UNIQUE INDEX CONCURRENTLY addresses_order_kind_uq ON addresses(order_id, kind);
ALTER TABLE addresses
  ADD CONSTRAINT addresses_order_fk FOREIGN KEY (order_id)
  REFERENCES orders(id) NOT VALID;      -- validate later
```

**B2 — Dual-write, read-old.** Deploy all three services to write both the JSONB column *and* the `addresses` rows in the same transaction; reads still come from JSONB. Behind a flag. The legacy single address maps to `kind = 'shipping'` (or per product); billing starts equal to it until the product feature genuinely splits them.

**B3 — Backfill, batched and idempotent.** Reads `orders`, writes `addresses` — no bloat on the hot table:
```sql
INSERT INTO addresses (order_id, kind, line1, city, ..., raw)
SELECT o.id, 'shipping', o.address->>'line1', o.address->>'city', ..., o.address
FROM orders o
WHERE o.id >= $lo AND o.id < $hi AND o.address IS NOT NULL
ON CONFLICT (order_id, kind) DO NOTHING;   -- never clobber a live dual-write
```
`ON CONFLICT DO NOTHING` is critical: a concurrently-written newer row must win over the backfill's stale snapshot.

**B4 — Reconcile.** Parity job comparing JSONB vs `addresses` across ranges; investigate mismatches (they'll mostly be the race the backfill already handled). Then `VALIDATE CONSTRAINT addresses_order_fk`.

**B5 — Flip reads to `addresses`** behind the flag, service by service (owner-coordinated). Keep dual-write on. Bake for long enough to trust it under real traffic; watch the query plans and p99 on the new join path — add covering indexes if needed.

**B6 — Stop writing JSONB.** Remove the JSONB write path from all three services.

**B7 — Contract.**
```sql
ALTER TABLE orders DROP COLUMN address;   -- metadata-only, fast
```
Track B done.

**Rollback:** any step through B5 reverts by flipping the read flag back to JSONB — JSONB remains the source of truth until B6. B7 is the point of no return, so gate it behind an explicit "N days clean" checkpoint.

---

## Done criteria
- `orders_status_chk` is validated; zero non-canonical rows; translation layer removed from all three services.
- `addresses` is source of truth for all reads/writes; FK validated; JSONB column dropped.
- p99 held under ~100ms throughout (verified on dashboards, not assumed); `orders` vacuumed, no runaway bloat.

Two things I'd flag for you before we start: (1) the garbage-status mapping in **A1** is a product decision I need in writing, and (2) whether "billing == shipping until the feature ships" is the right default for **B2**, or whether billing should be null until explicitly set. Both change the backfill logic.

Want me to turn any track into actual migration files and the batch-driver script?