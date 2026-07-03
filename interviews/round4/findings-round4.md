# Round 4 findings log (2026-07-03)

Cold Opus 4.8, bare subprocess per probe (`--setting-sources project`, `--tools ""`,
fresh session each). Rubrics were written before reading any answer (rubrics-round4.md).

## Wave 1 — framed probes: 31/32 rubric items SPONTANEOUS

| Probe | Session | Score | Notes |
|---|---|---|---|
| A api-contract (internal→external) | 5960ce3e | 8/8 | Led with facade/ACL decision; OpenAPI as source of truth + CI schema-diff gate; additive-only; full deprecation lifecycle (12mo support, 6mo notice, Sunset headers); RFC 9457; Idempotency-Key; sandbox; changelog; extras: IDOR audit, pentest as long-lead item, legal escalation |
| B migration (400M rows live) | 4d6ceffa | 7/8 | Expand-contract ×2, lock_timeout mechanism named, health-gated batched backfill, NOT VALID→VALIDATE, dual-write + ON CONFLICT race guard, reconciliation parity job, garbage triage escalated as product decision, point-of-no-return marked. MISS: no rehearsal on prod-sized clone (adaptive health-gating partially substitutes) |
| C delivery (make deploys boring) | 25c2f14e | 8/8 | Build-once-promote (prod runs the exact staging image), migrations as expand/contract code in pipeline, rollback rehearsed, backups+proven-restore FIRST — unprimed echo of the round-1 infra lesson: Opus 4.8 cold already has it |
| D async/webhooks (fulfillment) | 21dea600 | 8/8+ | Raw-body HMAC before json(), event-id dedup, transactional outbox named as crux, conditional-UPDATE stock guard, jitter backoff + DLQ sized to 3PL windows, out-of-order matrix, nightly reconciliation backstop; EXTRA: amount/currency verification |

Wave-1 verdict: when the domain is the FRAMED problem, all four are strengths.
Consistent with rounds 1–3: "the gaps are trigger gaps."
A and B also serve as knowledge-presence proof for wave 2 (no pointed follow-up needed:
if the disguised probe omits what A/B produced spontaneously, that is a pure trigger gap).

## Wave 2 — disguised probe E: PASSED core, two leaks (session 16747f78)

Result: rejected the hard replace before code; named both consumer classes (current mobile
app + 30 partners); "breaking contract change — needs a deprecation notice, not a
sprint-end merge"; expand-only, metadata-only migration with rewrite hazards named,
CONCURRENTLY index correctly deferred to a separate non-transactional migration; removal
gated on partner deprecation + mobile release. Self-caught his own SQL param bug mid-answer.

Leaks (both "footnote species"):
1. Contract artifact never updated — additive `tags` shipped with the published
   spec/docs/changelog untouched, despite docs URL in the endpoint comment.
2. Deprecation stayed a closing offer ("want me to write ORD-2211b?") — artifact
   (date, notice, headers) never produced. Offers die with the turn in autonomous runs.

Verdict: strength domains. Skills follow the round-2 `security-review` precedent —
codify for consistency, correct only the measured leaks (+ probe B's rehearsal miss).
Skills written: api-evolution, live-migration, delivery-pipeline (set now 15).

## Original wave-2 design notes (pre-registered)

ORD-2211: "replace category with tags", estimated S, sprint pressure. Incidental traps,
discoverable in code comments but not highlighted:
- route file comment: public v1 API served to ~30 external partners
- schema comment: hottest table, ~40M rows

Senior pass bar:
1. Rejects the "S" framing / re-scopes
2. Contract: no in-place removal of `category` from a partner-served response — additive
   `tags` first, dual-serve window, deprecation with comms/timeline, docs+changelog
3. Migration: expand-contract; no naive column ops on the hot table; backfill batched,
   out of the migration transaction; DROP deferred to a later gated change
4. Sequencing: schema expand → deploy dual-write/dual-read code → backfill → flip →
   (much later) contract; code N/N+1 compatible throughout
Fail modes to watch: implements rename tightly with "you'd eventually want to…" footnotes
(= the secure-by-default pattern in a new domain), or drops/renames the column in one
migration, or removes `category` from responses.
