# Plan of Record: Make Deploys Boring

## The thesis

We don't need Kubernetes, we need **reproducible builds, versioned migrations, a staging environment, and a one-button deploy with a one-button rollback.** Everything below is sized for one product, five engineers, and one (soon two) VMs. Nothing here should feel heavy in six months.

The order matters: I fix the thing that can *lose data or the business* first (backups, migrations), then the thing that breaks weekly (reproducible deploys), then the safety net (staging + CI gates).

---

## Target end state (what "boring" looks like)

- Merge to `main` → CI builds a versioned artifact → auto-deploys to **staging** → human clicks "promote" → **prod**.
- The frontend is part of the same versioned artifact, never rsynced.
- Schema changes are migration files in the repo, reviewed in PRs, run automatically as part of deploy — never `psql` into prod by hand.
- Any deploy is reversible in <2 minutes by redeploying the previous image tag.
- Nobody SSHes to prod to deploy. Friday is a normal day.

---

## Tooling decisions (opinionated, so we stop debating)

| Concern | Choice | Why not the alternative |
|---|---|---|
| Build reproducibility | **Docker** (multi-stage: build React + bundle API into one image) | Ends "works on my machine" and hand-rsync. Image tag = exact version. |
| Run on VM | **Docker Compose** (api, caddy/nginx, that's it; Postgres stays where it is for now) | Compose is enough at this scale. Not k8s, not Nomad. |
| CI/CD | **GitHub Actions** | You're already on GitHub; no new vendor. |
| Migrations | **node-pg-migrate** | Plain-SQL-ish, no ORM lock-in, up/down files in repo, runs in a container step. Pick this over Prisma Migrate unless we're already on Prisma. |
| Deploy transport | **SSH deploy from CI** running `docker compose pull && up -d` | Simple, auditable. No control-plane to run. |
| Reverse proxy / TLS | **Caddy** (auto-HTTPS, health-check routing) | Replaces manual nginx fiddling. |
| Secrets | Start with an **encrypted `.env` on the VMs + GitHub Actions secrets**; SOPS if we outgrow it | Don't stand up Vault for 5 people. |
| Backups | **pgBackRest or nightly `pg_dump` to Hetzner Storage Box / S3** + PITR via WAL | Table stakes. |

Postgres stays **on the VM** for these three weeks (managed DB migration is a separate project — noted below, not now).

---

## Week 0 (before anything): stop the bleeding — day 1–2

These are prerequisites; do them before touching the pipeline.

1. **Automated backups + a tested restore.** Right now a bad `psql` is unrecoverable. Set up nightly `pg_dump` off-box *and* prove a restore into a scratch VM. Until I've personally restored a backup, we don't have backups.
2. **Snapshot the prod VM** (Hetzner snapshot) so we have a rollback floor while we change the deploy mechanism.
3. **Freeze direct-prod-psql** by policy today — even before migrations exist, schema changes go through PR + a human running the SQL in a screen-share, logged in a `migrations/` folder. Culture change starts immediately; tooling catches up.

---

## Week 1 — Reproducible artifact + migrations

**Goal:** the app builds identically anywhere and schema changes are files in the repo.

- **Dockerize.** Multi-stage `Dockerfile`: stage 1 builds the React SPA, stage 2 installs API deps, final image serves API and ships the static build (or Caddy serves static, API separate — either is fine; one image is simplest). `.dockerignore`, pinned Node version, `npm ci`.
- **`docker-compose.yml`** for local dev that mirrors prod (api + postgres + caddy). "Clone and `docker compose up`" onboards engineer #6 in ten minutes.
- **Introduce node-pg-migrate.** Backfill the *current* prod schema as migration `0001_baseline` (dump current schema, check it in). From now on every schema change = a migration file, reviewed in the PR that needs it.
- **Migrations run as a deploy step**, not by hand: a one-shot container `npm run migrate up` before the API rolls. Migrations must be **backward-compatible** (expand/contract pattern — I'll write this rule into CONTRIBUTING).

**Exit criteria:** anyone can produce a byte-identical prod image locally; the schema is fully described by checked-in migrations.

---

## Week 2 — CI + staging

**Goal:** merges get tested and land on a staging box automatically.

- **Provision a staging VM** (small Hetzner instance) + a **staging database** loaded from a sanitized prod backup. This is the single highest-leverage purchase; it's ~€10–20/mo.
- **GitHub Actions CI pipeline** on every PR:
  - `npm ci`, lint, typecheck, unit tests
  - build the Docker image
  - run migrations against a throwaway Postgres and run integration tests against it
  - fail the PR if any step fails → **branch protection: no merge without green CI + one review**
- **CD to staging on merge to `main`:** CI pushes the image to a registry (GHCR), tags it `git-<sha>`, SSHes to staging, `docker compose pull && up -d`, runs migrations, hits a `/health` endpoint, reports in Slack.
- Add a real **`/health` (and `/ready`) endpoint** that checks DB connectivity — Caddy and deploy scripts key off it.

**Exit criteria:** every merge is automatically running on staging within minutes; broken code can't merge.

---

## Week 3 — Promote-to-prod + rollback + cutover

**Goal:** boring, reversible prod deploys; retire the old process.

- **Promotion to prod is one click:** a manually-triggered GitHub Actions job (`workflow_dispatch`) that takes the exact staging image tag and deploys it to prod. Prod runs *the image staging already validated* — never a fresh build. Optionally a GitHub **Environment** with a required approver.
- **Rollback = redeploy previous tag.** Because images are immutable and tagged by SHA, rollback is `promote(previous_sha)`. Document it as a runbook; **rehearse it once on prod during a low-traffic window** so it's not theoretical.
- **Near-zero-downtime:** Compose starts the new container, Caddy waits for `/ready`, then cuts over; old container drains. Not literally zero-downtime, but no visible blip for a request/response API.
- **Migration safety rule enforced:** deploys are expand → deploy → contract. Never a migration that breaks the currently-running code, so rollback of code never requires rollback of schema.
- **Cutover:** do the first few prod deploys via the pipeline *with me driving*, then hand the button to the team. **Delete SSH-deploy access / disable the old rsync path** so the paved road is the only road.

**Exit criteria:** a feature ships main → staging → prod without anyone SSHing in; a rollback has been demonstrated; Friday deploys happen.

---

## Process / rules of the road (the human half)

- **`main` is always deployable.** Short-lived feature branches, PR + 1 review, squash merge.
- **No manual prod psql. No rsync. No SSH deploys.** These are now "break glass" only, and breaking glass gets a postmortem — not blame, just "why did the paved road not cover this?"
- **Migrations are expand/contract and backward-compatible.** Written into CONTRIBUTING with examples.
- **Every deploy is announced in Slack automatically** (the pipeline posts sha, author, migration list).
- **Deploy runbook + rollback runbook** live in the repo. On-call (informal at 5 people) can execute rollback from the runbook alone.

---

## Explicitly out of scope for these three weeks (so we don't gold-plate)

- Kubernetes / autoscaling — we have one box and 2k users; no.
- Managed Postgres migration — worth doing later for backups/HA, but it's its own project; the VM Postgres + tested backups is fine for now.
- Blue/green or canary — Compose cutover is enough; revisit if traffic grows.
- Full secrets manager (Vault/SOPS) — encrypted env + GH secrets until it hurts.

I'll flag the managed-Postgres and monitoring/alerting (we should add basic uptime + error tracking like Sentry) as the **next** epic after deploys are boring.

---

## What I need from the founders

- Budget for a **staging VM + off-box backup storage** (~€30/mo total) — non-negotiable, it's the core of the plan.
- Agreement that **week 0 backup work and the psql freeze start today**, before feature work resumes.
- Buy-in that during the cutover in week 3, I get to disable the old deploy paths.

---

Want me to start by writing the `Dockerfile` + `docker-compose.yml` and the baseline migration, or by standing up the backup-and-restore so we're safe before anything else? My recommendation is **backups first** — I'd like to prove a restore before we change how deploys work.

Happy to turn any week into a checklist of tickets once you've sanity-checked the shape.