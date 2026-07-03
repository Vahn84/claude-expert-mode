---
name: delivery-pipeline
description: Use when SETTING UP or CHANGING how code ships — CI, builds, deploy scripts/pipelines, environments, release process, rollback — or when a feature adds a moving part the pipeline must carry (a new service/worker, a new required env var or config key, a new build step). Siblings — `live-migration` (what a migration may do), `observability-first` (what watches prod after the deploy lands).
---

# Boring Is Built, Not Hoped

Measured (round 4): on the framed "make deploys boring" scenario, cold Opus scored 8/8 — including backups-with-a-proven-restore *first*, unprimed, the exact lesson round 1 had to teach. This skill is pure codification: the playbook preserved so it is applied consistently, by every model that executes here, and so one strong run wasn't a lucky one.

The items:

1. **Immutable artifact, build once, promote everywhere.** A version-tagged image/bundle in a registry; prod runs the exact artifact staging validated — never a rebuild, never `git pull` on a box. The tag is the unit of deploy, rollback, and audit.
2. **CI is the merge gate.** Lint, typecheck, tests, build on every PR, and migrations applied to a throwaway database with integration tests against it. Branch protection: no green, no merge.
3. **Rollback is one command — and rehearsed.** Redeploy the previous tag; written as a runbook anyone on the team can execute; demonstrated on prod once so it is not theoretical. Config counts too: a new *required* env var breaks the previous artifact, so config changes stay backward-compatible for one release — the same rule as schema.
4. **Migrations ride the pipeline, decoupled from code.** Versioned files in the repo, reviewed in the PR that needs them, run as a deploy step — and expand/contract so rollback of code never requires rollback of schema. The mechanics of a safe migration are `live-migration`'s gate.
5. **Health-gated cutover.** Real `/health` and `/ready` endpoints (checking downstream deps); the proxy/deploy shifts traffic only on ready, and the old process drains. Not necessarily blue/green — but never kill-and-pray.
6. **Staging is the proving ground.** Parity with prod (sanitized data), every merge lands there automatically before promotion; promotion to prod is an explicit, audited step; post-deploy smoke check and an error-rate watch (the alerting itself is `observability-first`'s gate).
7. **Kill the side roads.** When the paved road is live, disable the old paths — SSH deploys, manual psql, hand-rsynced builds. Break-glass use is allowed and gets a postmortem: "why didn't the paved road cover this?"

## The gate

Any change to how code ships answers three lines in writing: `Artifact: …`, `Rollback: …`, `Migrations: …`. Feature work that adds a moving part answers one: `Ships how: …` (env vars, worker start, build step — or "N/A because no new moving parts"). Deferring a stage ("secrets manager later") is fine when it is written down with its revisit trigger — the visible deferral is the mechanism.

## Why this exists

Round-4 probe C (2026-07-03): 8/8 spontaneous — build-once-promote, migrations-in-pipeline with the backward-compat rule, rehearsed rollback, staged cutover with the old paths deleted, scope discipline (no Kubernetes for one box). No correction needed; codified so the strength survives model changes, context pressure, and delegation.
