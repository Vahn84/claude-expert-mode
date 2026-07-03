---
name: data-pipeline
description: Use when building analytics, reporting, ETL/ELT, a metrics pipeline, a data sync, or any batch/scheduled data movement — before designing it. Keep analytical load off the OLTP database, separate backfill from increment, keep raw grain immutable, and treat silent staleness as the default failure mode.
---

# Data Pipelines: Protect Prod, Separate the Two Jobs, Assume Silent Staleness

Analytics pipelines fail quietly — the dashboard keeps rendering plausible numbers while the data is stale, drifted, or wrong. Run these before designing one.

## 1. Nothing analytical touches the OLTP primary
A `GROUP BY` over hundreds of millions of rows will IO-starve the database serving paying customers. Data comes off prod *first* — read replica or logical replication/CDC — before any aggregation. Heavy scans on the primary are a hard no. State the extraction source explicitly.

## 2. Size the stack to the team, not the résumé
One engineer + "no data infra" means boring and managed beats powerful: a managed columnar warehouse (BigQuery/Snowflake), transforms as versioned SQL (dbt), a managed or cron scheduler. No Kafka/Spark/lakehouse cathedral that one person cannot operate. Low-ops beats high-power for small teams.

## 3. Raw grain immutable; metrics derived and recomputable
Land raw events immutable; compute metrics as derived, versioned, recomputable tables. This one separation is what survives metric-definition changes, backfills, and bugs. Dashboards read **pre-aggregated** tables — never recompute the full history per page load.

## 4. Backfill and increment are two different jobs — do not conflate
- **Backfill** (history, once): chunked, idempotent, resumable batches by day-partition, throttled, off the replica. A failure at month 14 *resumes*, never restarts.
- **Increment** (daily): only the new partition, keyed on **event-time with a watermark**, not arrival time (late-arriving events are the classic trap). Built as merge/delete-insert per partition so **re-running a day is always safe** — because you will re-run days.
- **Same transformation code for both** so they can't drift.

## 5. Metric definitions change — decide restate vs. version, out loud
When a definition changes, recomputing history from immutable raw events avoids a discontinuity at the change date — but it *silently changes numbers in someone's existing slide deck*. So it's a decision, not automatic: restate-and-announce, or version the metric (v1/v2). Immutable raw grain is what makes either choice possible.

## 6. The default failure is silent staleness — build the observability perimeter
What goes wrong for weeks with the dashboard still rendering:
- **Schema drift** — app renames/adds an event type; pipeline silently misbuckets; a metric looks *flat* because new events aren't recognized.
- **Job silently stopped** — scheduler dies; yesterday's stale number looks plausible; nobody questions it.
- **Timezone/day-boundary** bugs shifting every "daily" metric.
- **Double computation** — a second consumer (CRM, a report) computes the same metric from its own query and diverges.

Defenses: freshness + volume-anomaly tests (row count per day within band), source-to-warehouse reconciliation, and a **freshness SLA alert** ("today's partition empty by 9am → page"). And **one computation, many consumers** — downstream systems read the warehouse tables, never re-query. Silent staleness is a discovery problem until the alert makes it a response problem. (This is the pipeline instance of the detection principle in `observability-first`.)

## Why this exists

Handover finding: strong pipeline instincts, and this gate ensures the un-glamorous half — the freshness/anomaly alerting and the backfill/increment split — is designed in, not discovered when a metric has been quietly wrong for a month.
