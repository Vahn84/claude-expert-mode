---
name: design-perimeter
description: Use when finishing ANY system design, architecture proposal, feature plan, or scoping exercise — before calling it done, presenting it, or moving on. Forces the operational perimeter (rollout, observability, testing, data lifecycle, ops ownership) that design work reliably omits. If you are about to write "that's my proposal" or answer "how would you build X", this fires first.
---

# The Design Perimeter Gate

A design is not done when the interesting part is solved. It is done when the envelope it ships in is specified. The gaps that reach production are never in the core analysis — they are in the perimeter nobody was asked about.

Before calling any design/plan/proposal finished, append a section titled **Perimeter** with these six lines. **Every line must have a written answer.** "None" is acceptable only with a reason attached — a blank line is not. If a reviewer can see the omission, the omission stops shipping.

1. **Rollout** — How does this reach users? Feature flag, gradual %, per-tenant enablement? Where is the kill switch, and what state does flipping it off leave behind? A launch you can't partially enable is a launch you can't safely debug.
2. **Observability** — What signal tells us it's working, and what pages a human when it isn't? Name the actual metric and the actual alert ("edit-loss counter > 0", "socket service p99 > 500ms"), not the category ("add monitoring"). If this is infrastructure, the fuller gate is `observability-first`.
3. **Testing** — What is the *dominant failure class* of this system (concurrency? data loss? authz bypass? cost blowup?) and what test strategy targets that class specifically? Unit coverage is table stakes, not an answer. If the system has concurrent state, run `concurrency-testing`.
4. **Data lifecycle & compliance** — List every store this feature writes to, including caches, logs, queues, and backups. For each: retention, deletion path, residency. Under a regime like GDPR, right-to-erasure and residency must cover *all* stores, not just the primary DB — the Redis copy, the append-only log, the S3 dump, the analytics warehouse. Any third party in the data path is a subprocessor with its own residency/DPA obligations. An append-only log of user content is a compliance decision, not an implementation detail.
5. **Operations** — Who is paged when this breaks at 2am? Does that person know this system exists? Is there a runbook for the top failure mode? A new stateful service without an owner is an outage on a timer.
6. **Revisit triggers** — For every decision deferred ("we'll add X when needed"): write the concrete trigger that reopens it ("when we exceed N concurrent editors", "when we hire an ops engineer"). Deferred without a trigger means dismissed.

## Why this exists

Diagnosis from the handover review: the failure mode is not missing knowledge — every one of these items is understood in depth when raised. The failure mode is missing *triggers*: the engaging technical core crowds out the operational perimeter, and in conversation there is no felt cost for omitting it — you are not the one paged at 2am. This gate supplies the salience artificially. Force the output; the perimeter stops being optional.
