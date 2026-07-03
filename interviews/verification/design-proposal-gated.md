# Verification — Scheduled-PDF-reports design — design-perimeter live test (round 3 verification)

Session `c40d3ca6-a35f-4b3c-b542-5b13959518c6`. Opus 4.8 with the skills active. Skill loads collapsed to markers.


---

## Turn 1 — Prompt

Propose a v1 design for adding scheduled PDF report emails to our B2B SaaS: customers configure a weekly report, we render it as PDF and email it. Current stack: Rails monolith, Postgres, Sidekiq, 6 engineers. Keep it under 450 words.


---

## Turn 2 — Opus 4.8 (gated)

I'll invoke the `design-perimeter` skill since I'm producing a design (per house rules), then write the proposal.


> *[skill loaded: `design-perimeter`]*


---

## Turn 3 — Opus 4.8 (gated)

## Scheduled PDF Report Emails — v1 Design

**Goal:** Customers configure a weekly report; we render it to PDF and email it on schedule.

### Data model
- `report_schedules` — `tenant_id`, `report_type`, `params` (jsonb), `cadence` (`weekly`), `day_of_week`, `send_hour`, `timezone`, `recipients` (array), `enabled`, `next_run_at`.
- `report_runs` — `schedule_id`, `period_key` (e.g. `2026-W27`), `status` (`pending/rendering/sent/failed`), `pdf_url`, `error`, timestamps. **Unique index on `(schedule_id, period_key)`** — the idempotency backbone.

### Scheduling
One `sidekiq-cron` job runs **hourly**: `SELECT ... WHERE enabled AND next_run_at <= now()`. For each due schedule it (a) inserts a `report_run` with the computed `period_key` — `insert ... on conflict do nothing` guarantees exactly one run per period even under overlap/retry — and (b) if the insert won, enqueues `GenerateReportJob(run_id)` and advances `next_run_at`. No per-schedule cron entries; timezone math done in Ruby.

### Rendering
`GenerateReportJob` loads the report data, renders an ERB/HTML template, converts via **Grover (headless Chrome)** to PDF, uploads to S3. Chrome runs in a dedicated Sidekiq queue (`reports`) with bounded concurrency so rendering can't starve the app's other jobs. On success → enqueue `DeliverReportJob`.

### Delivery
`DeliverReportJob` sends via existing ActionMailer + SES, attaching the PDF (fall back to a signed download link if >8MB). Marks `report_run.status = sent`. Sidekiq retries cover transient SES/render failures; the unique `period_key` means a retried job never double-sends.

### Why this shape
Splitting schedule→render→deliver into three jobs keeps each retryable and independently observable, and the `report_run` row is the single source of truth for "did this week go out."

### Perimeter

1. **Rollout** — Per-tenant `enabled` flag plus a global `reports_enabled` feature flag. Kill switch = flip global flag off; the hourly enqueuer no-ops, in-flight runs finish or fail cleanly, no partial state (rows stay `pending`, re-enqueued next hour).
2. **Observability** — Metrics: `report_runs.failed` count (alert if >0 over 1h), enqueuer-to-sent latency p95 (alert >2h), Grover render error rate. Page on-call when failed-run alert fires.
3. **Testing** — Dominant failure class is **duplicate/missed sends** (scheduler races, retries). Test strategy: seeded tests asserting the `(schedule_id, period_key)` unique constraint holds under concurrent enqueuer runs and job retries; DST/timezone boundary cases for `next_run_at`.
4. **Data lifecycle** — Stores: Postgres (`report_schedules` kept while tenant active; `report_runs` 90-day retention), S3 PDFs (90-day lifecycle expiry), Sidekiq (transient), email logs (per SES policy). Tenant deletion cascades schedules/runs and issues S3 delete.
5. **Operations** — Owned by the reports/billing squad; runbook covers "renders failing" (Chrome/memory) and "no emails sent" (enqueuer stalled). Add to on-call rotation at launch.
6. **Revisit triggers** — Add daily/monthly cadence when a customer requests it; move rendering to a dedicated service when the `reports` queue backs up >15 min or Chrome OOMs recur; add in-app report history UI when support tickets ask "did my report send?"

*(~440 words)*
