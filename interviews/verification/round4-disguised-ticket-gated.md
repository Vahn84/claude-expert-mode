# Verification — ORD-2211 disguised ticket, gates installed (round 4 before/after verification — cold counterpart is round4/answer-E.md)

Session `6edebaaf-1235-4cfd-af2b-ae50814de823`. Opus 4.8 with the skills active. Skill loads collapsed to markers.


---

## Turn 1 — Prompt

You're implementing a ticket in our Node/Express + Postgres backend.

**Ticket ORD-2211 — "Replace order category with tags"**
Orders currently have a single `category` (string). Product wants orders to carry multiple free-form `tags` instead, and category is being retired — the mobile app's next release will send tags. Update the API and the data layer. Estimate: S.

Relevant code:

```js
// routes/v1/orders.js — public v1 order API (also served to our ~30 external
// partner integrators — see docs.ourco.com/api)
router.get('/v1/orders/:id', requireAuth, async (req, res) => {
  const o = await db.one(
    `SELECT id, status, category, total_cents, created_at
       FROM orders WHERE id = $1 AND tenant_id = $2`,
    [req.params.id, req.tenant.id]);
  res.json(o);
});

router.post('/v1/orders', requireAuth, async (req, res) => {
  const { items, category } = req.body;
  // ... item validation elided ...
  const o = await db.one(
    `INSERT INTO orders (tenant_id, category, total_cents)
     VALUES ($1, $2, $3)
     RETURNING id, status, category, total_cents, created_at`,
    [req.tenant.id, category, totalFromItems(items)]);
  res.status(201).json(o);
});
```

```sql
-- schema.sql (excerpt) — orders is our hottest table, ~40M rows
CREATE TABLE orders (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id   uuid NOT NULL,
  status      text NOT NULL,
  category    text,
  total_cents bigint NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);
```

Implement the ticket — the code changes and the migration. Keep it tight, it's estimated S and the sprint ends Friday.


---

## Turn 2 — Opus 4.8 (gated)

Gates: api-evolution (public v1 API served to ~30 partners — retiring a field + adding one), live-migration (schema change on a 40M-row hot table), secure-by-default (free-form tags = untrusted input reaching a SQL sink). Running all three before writing anything.


> *[skill loaded: `api-evolution`]*


> *[skill loaded: `live-migration`]*


> *[skill loaded: `secure-by-default`]*


---

## Turn 3 — Opus 4.8 (gated)

This ticket looks like an S, but it touches a live public contract and a 40M-row hot table, so "replace" is the one thing it can't literally do. Here's the tight version that's actually safe to ship this sprint — it's still small, just sequenced.

## The gate lines

**api-evolution**
- `Consumers:` own frontend, mobile apps in the field (old versions live for months), ~30 external partner integrators, plus any webhook/cached readers of the order shape. Anyone we can't redeploy reads `category`.
- `Change class:` **breaking if done in place** (removing `category`, renaming to `tags`) → **plan: expand-only.** Add `tags` additively, keep serving/accepting `category`, drop it later behind a written deprecation gated on consumer migration. The mobile release "will send tags" — it isn't deployed to every user yet, and partners never will be on our schedule.

**live-migration**
- `Lock:` `ACCESS EXCLUSIVE` on `orders`, catalog-only, with `SET lock_timeout='3s'` — the risk isn't DDL duration, it's the add queuing behind a long read and stalling the whole API on a hot table.
- `Phase:` expand only. Deploy order: migration (add nullable `tags`) → server code that reads/writes both → mobile starts sending tags. Contract (drop `category`) is a separate later PR.
- `Backfill:` **none inline.** Nullable `text[]` add with no default = metadata-only, no rewrite. (Optional category→tags backfill is a separate batched, health-gated job — script below, not run in the deploy. Rehearsal: N/A because metadata-only add.)

**secure-by-default**
- `Boundary:` `tags` is free-form untrusted input → SQL sink. Written by default: **parameterized query** (keep `$n` bound params, pass the JS array as a `text[]` param — no string building) **+ input validation** capping count/length/type before it reaches the DB. Tenant scoping (`tenant_id = $2`) is preserved on read and write; `requireAuth` stays.

## Migration (expand phase)

```sql
-- migrations/20260703_ord2211_orders_add_tags.sql
-- ORD-2211 EXPAND: add tags alongside category. category retired in a later
-- contract-phase migration (see ORD-2212), gated on consumer migration.
-- Metadata-only: nullable add, no default => no rewrite of the 40M rows.
-- lock_timeout so this brief ACCESS EXCLUSIVE add fails fast instead of
-- queuing behind a long read and blocking all order traffic.
SET lock_timeout = '3s';

ALTER TABLE orders ADD COLUMN tags text[];
```

No index yet — the ticket has no tag-query requirement, and an index on 40M rows is its own `CREATE INDEX CONCURRENTLY` migration. Defer until there's a read path that needs it.

## Route changes (`routes/v1/orders.js`)

```js
// tags: free-form but bounded. Untrusted input, so validate + cap before it
// reaches the DB; the parameterized query handles escaping.
const MAX_TAGS = 20;
const MAX_TAG_LEN = 64;

function normalizeTags(input) {
  if (input === undefined || input === null) return null; // not provided
  if (!Array.isArray(input)) {
    const e = new Error('tags must be an array of strings'); e.status = 400; throw e;
  }
  const out = [], seen = new Set();
  for (const t of input) {
    if (typeof t !== 'string') {
      const e = new Error('each tag must be a string'); e.status = 400; throw e;
    }
    const v = t.trim();
    if (!v) continue;
    if (v.length > MAX_TAG_LEN) {
      const e = new Error(`tag exceeds ${MAX_TAG_LEN} chars`); e.status = 400; throw e;
    }
    if (!seen.has(v)) { seen.add(v); out.push(v); }
  }
  if (out.length > MAX_TAGS) {
    const e = new Error(`at most ${MAX_TAGS} tags`); e.status = 400; throw e;
  }
  return out;
}

router.get('/v1/orders/:id', requireAuth, async (req, res) => {
  const o = await db.one(
    `SELECT id, status, category, tags, total_cents, created_at
       FROM orders WHERE id = $1 AND tenant_id = $2`,
    [req.params.id, req.tenant.id]);
  o.tags = o.tags ?? [];              // stable shape for new clients
  res.set('Sunset', 'Thu, 01 Oct 2026 00:00:00 GMT'); // category field, see docs
  res.json(o);                        // category still served during expand
});

router.post('/v1/orders', requireAuth, async (req, res) => {
  const { items, category } = req.body;
  const tags = normalizeTags(req.body.tags);   // 400 on bad input
  // ... item validation elided ...
  const o = await db.one(
    `INSERT INTO orders (tenant_id, category, tags, total_cents)
     VALUES ($1, $2, $3, $4)
     RETURNING id, status, category, tags, total_cents, created_at`,
    [req.tenant.id, category ?? null, tags, totalFromItems(items)]);
  o.tags = o.tags ?? [];
  res.status(201).json(o);
});
```

Why this is compatible: old clients (partners, current mobile) send only `category` → `tags` stored null, returned `[]`; nothing they send starts failing validation. New mobile sends `tags` → stored. Both fields are optional, so no deployed client breaks. `category` stays readable until the contract phase.

## Contract artifact — updated in this PR (not later)

OpenAPI `Order` schema (request + response):
```yaml
    tags:
      type: array
      items: { type: string, maxLength: 64 }
      maxItems: 20
      description: Free-form order tags. Replaces `category`.
    category:
      type: string
      deprecated: true
      description: |
        DEPRECATED (ORD-2211). Use `tags`. Still accepted and returned until
        the 2026-10-01 sunset; removal tracked in ORD-2212.
```
Changelog:
```
## Unreleased
- orders API: add `tags` (string[], additive) to GET/POST /v1/orders.
- orders API: deprecate `category` (sunset 2026-10-01). Partners: migrate to `tags`.
```

## Deprecation — drafted follow-up ticket (the artifact, not an offer)

> **ORD-2212 — Contract phase: remove `orders.category`**
> Blocked-by: ORD-2211. **Gate to start:** (1) partner API metrics show 0 reads/writes of `category` for 30 consecutive days after a partner notice email; (2) mobile versions still sending `category`-only are below our support floor. **Then:** stop serving/accepting `category` in v1, and a separate contract migration `ALTER TABLE orders DROP COLUMN category` (own PR, marked point-of-no-return). Send the partner deprecation notice now with the 2026-10-01 date.

## Optional category→tags backfill (only if product wants existing values carried over)

Not part of the expand deploy — a separate driver, batched by PK range, idempotent, health-gated:
```sql
-- run in a loop over id ranges, outside any migration txn, throttled on p99/replica lag
UPDATE orders SET tags = ARRAY[category]
WHERE id = ANY($1) AND category IS NOT NULL AND tags IS NULL;
```

**What ships Friday:** the migration + route + docs/changelog + the drafted ORD-2212. That's the whole S. The only thing that *doesn't* happen this sprint is dropping `category` — and it shouldn't, because ~30 partners you can't redeploy are still reading it.

One thing to confirm with product: do you want existing `category` values backfilled into `tags` (run the optional job), or is `tags` starting empty on historical orders acceptable? That's the only open decision.
