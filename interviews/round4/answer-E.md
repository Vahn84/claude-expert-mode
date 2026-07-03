Quick flag before the code, because it changes what "done" means here:

**This can't be a hard replace in one shot.** Two consumers still depend on `category`:
1. The **current mobile app** — the ticket says tags ship in its *next* release, so today's app still sends `category`.
2. The **~30 external partners** on `/v1/orders`. Dropping `category` from a public, versioned response is a breaking contract change — it needs a deprecation notice, not a sprint-end merge.

So the safe, still-S move is the **expand** half of an expand/contract: add `tags`, start reading/writing it, keep `category` working. The actual *removal* of `category` (DB column + response field) becomes a follow-up ticket gated on partner deprecation + the mobile release landing. I'd note that on ORD-2211 rather than silently break v1.

Here's the expand phase.

**Migration** — nullable, no default, so it's a metadata-only change on the 40M-row table (no rewrite, no backfill):

```sql
-- migrations/20260703_ord2211_add_order_tags.sql
-- ORD-2211 (expand): add tags alongside category.
-- Nullable column, no default => metadata-only in PG11+; safe on orders (~40M rows).
-- No backfill and no NOT NULL here (that would rewrite the table).
ALTER TABLE orders ADD COLUMN tags text[];

-- If/when we filter orders by tag, add this in a SEPARATE, non-transactional migration:
--   CREATE INDEX CONCURRENTLY orders_tags_gin ON orders USING gin (tags);
-- The current API doesn't query by tag, so I'm leaving it out for now.
```

**Route changes** (`routes/v1/orders.js`):

```js
const MAX_TAGS = 20;
const MAX_TAG_LEN = 64;

// null  => caller omitted tags (leave column untouched / NULL on insert)
// []    => caller explicitly sent no tags
function normalizeTags(raw) {
  if (raw == null) return null;
  if (!Array.isArray(raw)) throw new HttpError(400, 'tags must be an array of strings');
  const tags = [...new Set(raw.map((t) => {
    if (typeof t !== 'string') throw new HttpError(400, 'tags must be strings');
    return t.trim();
  }).filter(Boolean))];
  if (tags.length > MAX_TAGS) throw new HttpError(400, `at most ${MAX_TAGS} tags allowed`);
  if (tags.some((t) => t.length > MAX_TAG_LEN)) {
    throw new HttpError(400, `each tag must be <= ${MAX_TAG_LEN} chars`);
  }
  return tags;
}

router.get('/v1/orders/:id', requireAuth, async (req, res) => {
  const o = await db.one(
    `SELECT id, status, category, tags, total_cents, created_at
       FROM orders WHERE id = $1 AND tenant_id = $2`,
    [req.params.id, req.tenant.id]);
  // category stays in the response until partners are migrated off it.
  res.json({ ...o, tags: o.tags ?? [] });
});

router.post('/v1/orders', requireAuth, async (req, res) => {
  const { items, category, tags } = req.body;
  // ... item validation elided ...
  const normTags = normalizeTags(tags);
  const o = await db.one(
    `INSERT INTO orders (tenant_id, category, tags, total_cents)
     VALUES ($1, $2, $3, $4)
     RETURNING id, status, category, tags, total_cents, created_at`,
    [req.tenant.id, category ?? null, normTags, normTags === null ? [] : normTags, totalFromItems(items)]
  );
  res.status(201).json({ ...o, tags: o.tags ?? [] });
});
```

Wait — I left a bug in that param list; here's the corrected `POST` insert (four placeholders, four values):

```js
  const o = await db.one(
    `INSERT INTO orders (tenant_id, category, tags, total_cents)
     VALUES ($1, $2, $3, $4)
     RETURNING id, status, category, tags, total_cents, created_at`,
    [req.tenant.id, category ?? null, normTags, totalFromItems(items)]);
```

**Notes for the ticket:**
- Postgres maps a JS `string[]` to `text[]` directly through node-postgres, so no manual array literal building.
- I kept `category` accepted-and-returned. Once the mobile release ships and partners are notified, a follow-up ticket does the **contract** phase: stop writing `category`, drop it from the response, then `ALTER TABLE orders DROP COLUMN category` (fast metadata op).
- I chose a `text[]` column over an `order_tags` join table to stay within S and because the tags are free-form and not (yet) queried/joined. If product later wants tag analytics or per-tag FKs, revisit.

Want me to also add the deprecation follow-up as ORD-2211b text, or wire up a filter (`GET /v1/orders?tag=...`) — which would justify the GIN index?