---
name: observability-first
description: Use for ANY infrastructure plan, deployment change, reliability roadmap, migration, or ops budget exercise — before finalizing priorities. Detection comes before elegance; a failure you can't see is a discovery problem, not a response problem.
---

# Detection Before Elegance

The day the box dies must be a page, not a customer email. No infrastructure plan is complete — at any budget — without detection as an explicit, early line item. It costs a day and it changes the nature of every future failure.

## The day-one kit (before anything architectural)
Write these into every infra plan, priority ~1.5 — after "prove restores work", before everything else:
- **External uptime check** on every user-facing endpoint (something *outside* the box, so the box dying doesn't kill the alarm too).
- **Host alerts:** disk, memory, CPU sustained — the boring trio that predicts most single-box deaths hours in advance.
- **Application alerts:** error rate and latency at the edge (nginx/ALB), not just inside the app.
- **Alert routing that reaches a human** — a Slack channel nobody watches is a log, not an alert. Name who gets paged.

## Backups are claims until restored
A backup that has never been restored is a hope, not a backup. In any plan that touches data:
1. **Restore test first, today, priority 1.** Restore the latest backup to a fresh environment. It is the cheapest, highest-value item on any infra list and it is almost always skipped.
2. **Write RPO and RTO as numbers.** "Nightly pg_dump" = RPO 24h — say so out loud and ask whether the business survives losing a day. Unmeasured RTO = assume the worst case. If either number is unacceptable, that outranks every elegance project on the roadmap.
3. **Schedule restore tests recurrently.** A restore that worked in January is a hope again by June.

> Related gates: this is the infra-plan expansion of `design-perimeter` #2 (Observability). For the analytics-pipeline flavor of the same failure — a dashboard rendering plausible-but-stale numbers — see `data-pipeline` #6 (freshness/anomaly alerting).

## Priority frame for constrained budgets
When someone (a CTO, a roadmap, you) proposes infrastructure modernization, sort by *risk reduction per dollar*, and separate the ask into outcomes — durability, availability, deployability, observability — before discussing tools. The tool conversation ("Kubernetes?") is deferred with a written revisit trigger, not dismissed. Managed services for stateful components (databases first) usually buy the most outcome per dollar for small teams.

## Why this exists

Handover-review finding: a full, otherwise-excellent infra quarter was planned — restore testing, managed Postgres, IaC, zero-downtime deploys — with monitoring appearing nowhere. In that plan, the company finds out the box died from angry customers. The analysis instinct optimizes for the failure you're discussing; this gate covers the failure nobody mentioned.
