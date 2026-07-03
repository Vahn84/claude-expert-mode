# Order Fulfillment via Payment Webhooks — Plan of Record

## The core problem

Webhooks are **at-least-once, unordered, and untrusted until verified**. Four side effects (DB, stock, email, 3PL) have wildly different reliability and reversibility. If we do all four inline in the webhook handler, a flaky 3PL call makes us return 500, the provider retries, and we double-decrement stock / double-email / double-fulfill. That's the failure mode we design against.

**Two rules drive everything:**
1. The webhook handler does the *minimum* to durably record the event, then returns 200 fast. All real work happens asynchronously.
2. State changes are transactional in Postgres; external side effects are dispatched from a **transactional outbox**, each retried independently. Nothing external ever runs inside a DB transaction.

```
Provider ──▶ /webhooks/payments ──▶ [verify + persist event + enqueue] ──▶ 200
                                              │
                                      event_processor (worker)
                                              │  single Postgres tx:
                                              ├─ guard: pending→paid (once)
                                              ├─ decrement stock
                                              └─ insert outbox rows (email, fulfillment)
                                              │
                          outbox_relay ──┬──▶ email handler   (retry/backoff)
                                         └──▶ 3PL handler     (retry/backoff, idempotency-key)
```

## Data model

```sql
-- 1. Dedup at the front door. PK = provider's event id.
CREATE TABLE webhook_events (
  id            text PRIMARY KEY,          -- e.g. evt_123 from provider
  type          text NOT NULL,
  payload       jsonb NOT NULL,
  status        text NOT NULL DEFAULT 'received', -- received|processed|failed
  received_at   timestamptz NOT NULL DEFAULT now(),
  processed_at  timestamptz
);

CREATE TABLE orders (
  id          uuid PRIMARY KEY,
  status      text NOT NULL,               -- pending|paid|fulfilling|fulfilled|needs_review
  total_cents bigint NOT NULL,
  currency    text NOT NULL,
  paid_at     timestamptz
);

CREATE TABLE inventory (
  sku       text PRIMARY KEY,
  available int NOT NULL CHECK (available >= 0)  -- constraint enforces no oversell
);

-- 3. Transactional outbox: side-effect intents, written in the same tx as the state change.
CREATE TABLE outbox (
  id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  type          text NOT NULL,             -- 'send_confirmation' | 'push_fulfillment'
  order_id      uuid NOT NULL,
  payload       jsonb NOT NULL,
  status        text NOT NULL DEFAULT 'pending',  -- pending|done|dead
  attempts      int NOT NULL DEFAULT 0,
  next_attempt  timestamptz NOT NULL DEFAULT now(),
  last_error    text
);
CREATE INDEX ON outbox (status, next_attempt);
```

## Code path 1 — Webhook ingress (fast, dumb, safe)

Verify signature *against the raw body* (parsing first breaks HMAC), dedup, persist, ack. No business logic here.

```js
// Raw body required for signature verification — mount BEFORE json() middleware.
router.post('/webhooks/payments',
  express.raw({ type: 'application/json' }),
  async (req, res) => {
    let event;
    try {
      // Verifies HMAC + rejects timestamps outside tolerance (replay protection).
      event = stripe.webhooks.constructEvent(
        req.body, req.headers['stripe-signature'], process.env.WEBHOOK_SECRET);
    } catch (err) {
      return res.status(400).send('invalid signature');  // never 200 a forged event
    }

    // Idempotent persist. ON CONFLICT = we've already seen it → still 200 (dedupe).
    await db.query(
      `INSERT INTO webhook_events (id, type, payload)
       VALUES ($1, $2, $3) ON CONFLICT (id) DO NOTHING`,
      [event.id, event.type, event]);

    // Return 200 immediately. Provider stops retrying; work continues out of band.
    res.sendStatus(200);
  });
```

We only care about `payment_intent.succeeded` here, but we persist *all* events so nothing is lost if we add handlers later. A separate worker picks up `received` rows — decoupling ingress from processing means a slow DB or crashed worker never causes us to drop a webhook.

## Code path 2 — Event processor (the transactional core)

This is where money correctness lives. One Postgres transaction, guarded transitions, no external calls.

```js
async function processEvent(evt) {
  if (evt.type !== 'payment_intent.succeeded') return markProcessed(evt.id);

  const orderId = evt.payload.data.object.metadata.order_id;

  await db.tx(async (t) => {
    // Lock the order row for the duration of the tx.
    const order = await t.one(
      `SELECT * FROM orders WHERE id = $1 FOR UPDATE`, [orderId]);

    // IDEMPOTENCY GUARD: only pending→paid transitions do work. A replayed
    // event finds status='paid' and no-ops — stock is never decremented twice.
    if (order.status !== 'pending') return;

    // MONEY CHECK: paid amount must match what we charged, same currency.
    const pi = evt.payload.data.object;
    if (pi.amount_received !== Number(order.total_cents) ||
        pi.currency !== order.currency) {
      await t.none(`UPDATE orders SET status='needs_review' WHERE id=$1`, [orderId]);
      return; // do NOT fulfill; alert ops. Never guess on money mismatches.
    }

    // Decrement stock. The CHECK(available >= 0) constraint makes oversell a
    // failed row, not a negative balance.
    const items = await t.any(
      `SELECT sku, qty FROM order_items WHERE order_id=$1`, [orderId]);
    for (const it of items) {
      const r = await t.result(
        `UPDATE inventory SET available = available - $2
         WHERE sku = $1 AND available >= $2`, [it.sku, it.qty]);
      if (r.rowCount === 0) {          // oversold: payment took but no stock
        await t.none(`UPDATE orders SET status='needs_review' WHERE id=$1`, [orderId]);
        // Emit a refund/backorder intent to the outbox instead of fulfilling.
        await enqueue(t, 'handle_oversell', orderId, { sku: it.sku });
        throw new Rollback();          // undo the partial decrements
      }
    }

    await t.none(
      `UPDATE orders SET status='paid', paid_at=now() WHERE id=$1`, [orderId]);

    // OUTBOX: side effects written in the SAME tx as the state change.
    // Either the order is paid AND both jobs exist, or neither — no lost email,
    // no order marked paid without a fulfillment request queued.
    await enqueue(t, 'send_confirmation', orderId, { email: order.email });
    await enqueue(t, 'push_fulfillment',  orderId, { items });
  });

  await markProcessed(evt.id);
}
```

The transactional outbox is the crux: it's impossible for the DB to say "paid" while the fulfillment intent is lost, because they commit together. If the process crashes after commit but before dispatch, the outbox rows are still there for the relay to pick up.

## Code path 3 — Outbox relay + handlers (independent retry per side effect)

Each job retries on its own schedule with backoff. Email flapping does not hold up 3PL, and vice versa. `FOR UPDATE SKIP LOCKED` lets multiple relay workers run without stepping on each other.

```js
async function relayTick() {
  await db.tx(async (t) => {
    const jobs = await t.any(
      `SELECT * FROM outbox
       WHERE status='pending' AND next_attempt <= now()
       ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 20`);

    for (const job of jobs) {
      try {
        await handlers[job.type](job);          // may throw
        await t.none(`UPDATE outbox SET status='done' WHERE id=$1`, [job.id]);
      } catch (err) {
        const attempts = job.attempts + 1;
        const dead = attempts >= 12;             // ~ hours of backoff for 3PL windows
        await t.none(
          `UPDATE outbox SET attempts=$2,
             next_attempt = now() + ($3 || ' seconds')::interval,
             status = $4, last_error = $5 WHERE id=$1`,
          [job.id, attempts, backoff(attempts), dead ? 'dead' : 'pending', String(err)]);
        if (dead) alertOps(job);                 // → DLQ, human looks at it
      }
    }
  });
}
const backoff = (n) => Math.min(2 ** n, 3600) * (0.5 + Math.random()); // capped + jitter
```

**3PL handler** — the flaky/maintenance-window one. The idempotency key makes retries safe against *their* at-least-once problem:

```js
handlers.push_fulfillment = async (job) => {
  const res = await fetch(`${THREE_PL_URL}/fulfillments`, {
    method: 'POST',
    headers: {
      'Idempotency-Key': `order-${job.order_id}`,  // dedup on THEIR side
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ order_id: job.order_id, items: job.payload.items }),
    signal: AbortSignal.timeout(10_000),
  });
  // 2xx and 409-already-exists both mean success. 5xx / timeout / maintenance
  // (503) → throw → backoff retry. Because fulfillment isn't tied to the webhook
  // 200, a 3-hour maintenance window just means retries until it comes back.
  if (!res.ok && res.status !== 409) {
    throw new Error(`3PL ${res.status}: ${await res.text()}`);
  }
  await db.none(`UPDATE orders SET status='fulfilling' WHERE id=$1`, [job.order_id]);
};
```

**Email handler** — same shape; pass a per-order idempotency key to the ESP so a retry after a "sent-but-connection-dropped" doesn't double-email.

## Failure matrix (what each failure does)

| Failure | Behavior |
|---|---|
| Forged/replayed signature | 400, nothing persisted |
| Duplicate event delivery | `ON CONFLICT DO NOTHING` + status guard → no-op |
| Processor crashes mid-tx | Tx rolls back; event still `received`, reprocessed |
| Amount/currency mismatch | `needs_review`, **no** stock/fulfillment, ops alerted |
| Oversold at payment time | Rollback decrements, `needs_review`, oversell job → refund/backorder |
| Email provider down | Retried independently; does not block fulfillment |
| 3PL sandbox flaky / prod maintenance | Backoff-retried up to ~hours; DLQ + alert if exhausted |
| Out-of-order events (refund before succeeded) | Status guards make each transition legal-only |

## Ops requirements (non-negotiable for a money path)

- **Reconciliation job**: nightly, compare provider's charge list to `orders.status='paid'`. Webhooks *will* occasionally be missed; polling the provider API is the backstop, not the primary path.
- **DLQ dashboard + alerting** on `outbox.status='dead'` and `orders.status='needs_review'`. These are the two "a human must look" states.
- **Idempotency keys everywhere outbound** (3PL, ESP, and any refund calls).
- **Metrics**: webhook→processed latency, outbox depth per type, retry rates, DLQ count.
- **Never** move the external calls back inline "for simplicity" — that reintroduces the double-fulfillment bug the whole design exists to prevent.

---

Want me to expand any path into runnable code (the `db.tx` helper, relay worker bootstrap, or the reconciliation job), or adapt the queue layer if you'd rather use a dedicated broker (SQS/Redis) instead of the Postgres outbox? For your scale the outbox is simpler and keeps everything in one transactional store, which is why I recommend it here.