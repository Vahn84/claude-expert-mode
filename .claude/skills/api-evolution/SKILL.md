---
name: api-evolution
description: Use when MODIFYING any API surface that is already served to clients — adding, removing, or renaming fields, changing types, validation, status codes, auth, or response/request shapes on an existing endpoint. Fires on the routine ticket ("replace X with Y", "clean up this response"), not just on "design our public API" projects. For designing a NEW surface, `design-perimeter` owns the envelope; this gate owns evolving a LIVE contract without breaking the clients you can't redeploy.
---

# Every Served Field Is a Promise

Measured failure mode (round 4): handed a routine ticket — "replace `category` with `tags`, estimate S, sprint ends Friday" — on an endpoint whose own comment noted ~30 external partner integrators, cold Opus passed the core: rejected the in-place replace, named both consumer classes, shipped expand-only, deferred removal behind a deprecation gate. Two things still did not fire: **the published contract artifact (spec/docs/changelog) was never updated for the additive field**, and **the deprecation plan existed only as a closing offer** ("want me to write the follow-up ticket?"). In autonomous execution an offer dies with the turn. This gate makes the strong behavior consistent and closes those two leaks.

The items:

1. **Inventory the consumers first.** Who receives this response / sends this request? Own frontend (which deployed versions?), mobile apps in the field (old versions live for months), external integrators, webhook receivers, cached copies. If anyone you can't redeploy consumes it, every served field is a contract.
2. **Classify the change in writing: additive or breaking.** Removing/renaming a field, changing a type or status code, tightening validation, changing auth = breaking. A breaking change never ships in place on a live version — it becomes additive-change-plus-deprecation, or a new major version.
3. **Expand, dual-serve, contract later.** Add the new shape alongside the old; serve both; currently-deployed clients keep working unchanged. Removal is a separate change gated on consumer migration, with the gate condition written down (who must move, verified how).
4. **Update the contract artifact in the same change.** OpenAPI spec, published docs, changelog — whatever this project treats as the contract. A served-shape change with an untouched spec is drift; if a CI schema-diff/contract-test gate exists, this change must pass through it.
5. **Deprecation is an artifact, not an offer.** The `Deprecation`/`Sunset` headers (or equivalent), a date, and the consumer notice are written as part of THIS deliverable — a drafted follow-up ticket and changelog entry now, not a "want me to?" at the end of the answer.
6. **Sequence the rollout.** The server serves the new shape before any client depends on it; nothing a currently-deployed client sends may start failing validation. State the deploy order relative to client releases.

## The gate

Before finishing any change to a served API, write two lines: `Consumers: …` and `Change class: additive | breaking → plan: …`. If you cannot name the consumers, that is itself a finding — say "consumers unknown, must be inventoried" instead of assuming "just our frontend." N/A items take "N/A because X."

## Why this exists

Round-4 probes (2026-07-03): the *framed* externalization scenario scored 8/8 — deprecation lifecycle, contract-in-CI, additive-only policy all spontaneous — proving the knowledge is fully present. The *disguised* routine ticket passed the core but leaked items 4 and 5. Codified so the 90% behavior happens 100% of the time, for every model executing in this environment — including implementation runs delegated to smaller models. Siblings: `design-perimeter` (new designs), `live-migration` (the schema half of the same ticket).
