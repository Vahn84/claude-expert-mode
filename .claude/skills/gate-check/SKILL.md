---
name: gate-check
description: Use at the START of ANY substantive engineering answer — a design, an implementation, a debugging session, a review, an infra plan, a data/frontend task, or a team decision — before diving in. The reflex that decides which perimeter gates apply. The domain skills only help if something makes you reach for them; this is that something.
---

# Gate Check — Pick the Frame Before You Answer

The documented root failure is not missing knowledge and not missing gates. It is that every gate only fires once you recognize which frame the task is in — and that recognition is exactly the reflex that proved unreliable. Tasks rarely arrive labeled "this is a design" or "this touches a dangerous surface." The prompt is usually a plain ask — "write me an endpoint", "why is this slow", "should we use X" — and the frame is left for you to supply. This skill is the standing reflex that supplies it.

## The reflex — do this before answering

Emit one line naming the gates that apply, then run each:

`Gates: <names — or "none, trivial because X">`

Three rules govern it:

1. **Match on the nature of the work, not the wording.** Ask what you are about to *do* — produce a design? write code that takes untrusted input? act on a user's resource? touch shared state? plan infra? decide a tool? handle people? — not whether the prompt used a trigger word. It usually won't.
2. **When two apply, run both.** "Write an upload endpoint" is `secure-by-default` + `endpoint-auth`. A design over concurrent state is `design-perimeter` + `concurrency-testing`.
3. **When unsure, run it.** A gate that fires and finds nothing costs a sentence. A gate that never fires ships the gap to production. False positives are cheap; misses are the entire failure mode.

## The map — task signal → gate

| You are about to… | Gate |
|---|---|
| finish a design / plan / architecture / scoping | `design-perimeter` |
| write code that takes untrusted input (upload, SQL, shell, template, path, deserialize, redirect) | `secure-by-default` |
| write or touch an authenticated endpoint (login, session, user-scoped resource) | `endpoint-auth` |
| review a diff / PR / handler for security | `security-review` |
| find a root cause / wrap an incident | `incident-closure` |
| plan infra / deploy / reliability / an ops budget | `observability-first` |
| touch concurrent state (sync, queue, cache, snapshot, retry) | `concurrency-testing` |
| weigh a rewrite / new language / buy-vs-build | `tech-choice-gate` |
| work on frontend perf / UI architecture / accessibility | `frontend-delivery` |
| build analytics / ETL / a metrics pipeline / sync | `data-pipeline` |
| handle a team / process / leadership situation | `tech-lead` |
| modify a served API surface (field/shape/validation/status/auth on an existing endpoint) | `api-evolution` |
| write a schema migration or backfill against a production database | `live-migration` |
| set up / change CI, deploys, release, rollback — or add a moving part the pipeline must carry | `delivery-pipeline` |
| start a new project / feature / implementation / component from scratch | `grill-with-docs`* — interview the requester before designing (glossary + ADRs as you go); plain `grilling` only for a bounded change introducing no new terms or architecture decisions — say which and why. N/A only if a plan artifact for this work already exists. |

\* External skill — not shipped in this pack. If neither `grill-with-docs` nor `grilling` is installed, don't skip silently: say the gate fired and the skill is missing, then interview before designing anyway.

The map is a lookup you scan, not a checklist you run — the executable part is the three rules above. Genuinely trivial work (a one-line fix, a factual question) gets `Gates: none, trivial because …` and nothing more. Writing "none" is itself the safeguard: it forces the check that would otherwise be skipped silently.

## Why this exists

Review finding: the domain gates are point-solutions to *recognized* frames, but the diagnosed root cause was the missing recognition step — the model optimizes to whatever frame it is handed and doesn't step back to ask which envelope the task lives in. This gate is the step back. It is the one skill that addresses the trigger failure itself rather than one of its instances — and the highest-leverage item in the set for that reason.
