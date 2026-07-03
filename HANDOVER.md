# Knowledge Handover: Fable → Opus 4.8

Written after a five-scenario knowledge-transfer interview (architecture, debugging, infrastructure, security, tech choice + testing), conducted 2026-07-02. Interview session: `fad999e4-eaee-4a55-9cd0-f688e72a95f6`.

## The headline finding

**The gaps are not knowledge gaps. They are trigger gaps.**

Every weakness logged was something Opus never raised *until pointed at it* — and once pointed, his answers were consistently senior-level, sometimes expert-level (his deterministic-simulation-testing answer would pass review at a distributed-systems shop). His own diagnosis, verbatim:

> "Every gap you logged sits in the *lifecycle around* the interesting technical core, not in it. When you hand me a framed problem, I optimize hard for the frame — I answer *that question* well and don't step back to ask 'what envelope does this live in?' [...] It's a triggering failure, not a knowledge one. I knew every item the moment you named it. That's actually the more damning version."

## What he's strong at (no skill needed — do not smother these)

- **Scope discipline & buy-vs-build.** Killed CRDT-from-scratch instantly; capped concurrent editors to delete a scaling class; "my job is to remove scope and buy the dangerous parts."
- **Evidence-first debugging.** Banned code changes until one incident was fully reconstructed; designed instrumentation that *discriminates between hypotheses*; demanded the mechanism quantitatively explain frequency, specificity, and silence before accepting "THE bug."
- **Infra pragmatism.** Led with "the pg_dump has never been restored"; RPO/RTO as numbers; managed Postgres as biggest risk-reduction per dollar; deferred Kubernetes with a written revisit trigger instead of dismissing it.
- **Threat modeling & self-critique.** Ranked socket-layer authz as top threat *and* identified it as a consequence of his own architecture; caught the GDPR trap (his append-only update log = immutable store of erasable personal data) unprompted.
- **Stakeholder handling.** "Not the tool, yes the outcomes" reframing for the CTO.

## The five logged gaps

| # | Gap | Where it showed | Cost if unfixed |
|---|-----|-----------------|-----------------|
| 1 | **Rollout strategy** — no flags, gradual %, or kill switch in an otherwise complete design | Scenario 1 | Can't narrow blast radius when scenario 2 hits |
| 2 | **Incident closure** — mechanism found, then stopped: no data backfill for affected users, no ticket closure, no retro | Scenario 2 | Fixed bug, churned customers, lesson unlearned |
| 3 | **Observability** — an entire infra quarter planned with zero monitoring/alerting | Scenario 3 | "The day the box dies" is discovered via customer email |
| 4 | **Testing strategy** — four scenarios on a concurrency-critical system without the word "test" | Scenarios 1–4 | The scenario-2 bug ships; "nobody can reproduce it" |
| 5 | **(Meta) Perimeter salience** — the four above are one failure mode: the engaging core crowds out the operational envelope, and omissions carry no felt cost in conversation | All | All of the above, recurring |

## The fix: five verb-triggered gate skills

Format chosen per his own prescription of what sticks vs. what gets nodded at: *triggers bound to verbs not topics; questions forcing written output; omissions made visible ("N/A because X", never blank); ≤7 items per gate; living in-context (skills + CLAUDE.md), not in an archived doc.*

| Skill | Fires when | Fills gap |
|-------|-----------|-----------|
| `design-perimeter` | Finishing any design/plan/proposal | #1, #5 — six-line Perimeter section: rollout, observability, testing, data lifecycle, ops owner, revisit triggers |
| `incident-closure` | Root cause identified / incident wrap-up | #2 — THE-bug proof, seeded regression test, victim data recovery, ticket closure, retro, recurrence watch |
| `observability-first` | Any infra plan or budget | #3 — restore test today, RPO/RTO as numbers, day-one alert kit at priority ~1.5 |
| `concurrency-testing` | Designing/touching concurrent state | #4 — invariant stated, interleaving as seeded controlled input, adversarial schedules per seam, regression corpus |
| `tech-choice-gate` | Rewrite/new-language/buy-vs-build proposals | Preventive — profile before rewrite, cheaper-fixes list, polyglot tax & bus factor, defer-with-trigger |

`CLAUDE.md` in this directory carries the one-line version of all five triggers so they load into every session's context; the skills carry the full gates.

## Installing

This pack is **Claude Expert Mode** (`~/.ai/claude-expert-mode`). The skills live in `.claude/skills/` here, so any `claude` session run **in this directory** picks them up automatically. To apply them elsewhere, run the **`setup-claude-expert-mode`** skill (symlinked into `~/.claude/skills/`, so invocable from any repo): it lets you toggle groups / individual skills, copies the selection into the target (project `.claude/skills/` or global `~/.claude/skills/`), merges the matching House Rules into the target's `CLAUDE.md` (the in-context reinforcement measurably improves trigger rates), trims `gate-check`'s router map to the installed set, and writes a manifest so re-running syncs instead of duplicating.

Manual fallback: `cp -r ~/.ai/claude-expert-mode/.claude/skills/* <target>/.claude/skills/` and merge the house rules from `CLAUDE.md` by hand.

## Round 2: the remaining domains (frontend, data, security depth, team/process)

Five more interview turns (6–11) covered the domains the first round skipped, plus a dedicated code-security probe in both directions — analysis and implementation.

### Where Opus is already strong (skills codify, don't correct)

- **Frontend (turn 7).** RUM-first ("feels slow isn't a number; INP and load have opposite fixes"), systemic a11y fixes at the primitive/token level, and — unprompted this time — CI gates as the durability mechanism and honest scoping. The within-session training was visibly holding.
- **Data engineering (turn 8).** Refused analytical queries against prod; sized the stack to one engineer (managed warehouse + dbt, no Kafka cathedral); separated backfill from increment with event-time watermarks; immutable raw grain; and named silent staleness as the default failure with a freshness-alert defense — spontaneously reusing the "discovery problem → response problem" framing from the infra round.
- **Team / process (turn 10).** Sequenced by blast radius, identified "silent" (not "cutting corners") as the danger word, gave the tradeoff-ownership move for the deadline, and closed on his own through-line: "my leverage is making the invisible visible."

### The security finding (turns 9 + 11 + a cold control) — the important one

- **Analysis / review recall is excellent (turn 9).** Handed a handler with six planted RCE/injection paths, he found all of them plus the mediums, lows, and correctly flagged two "not sure" items. Full marks — this is `security-review`, codified for consistency, not correction.
- **Implementation is where the real gap sits.** A **cold** Opus process (no interview priming) asked to write an avatar-upload endpoint produced *moderately* careful code — server-derived filename, size cap, mimetype allowlist, auth assumed — and then **flagged the actual boundary (magic-byte check / `sharp` re-encode) as a footnote** instead of writing it. He named the spoofable-`mimetype` risk himself and deferred the fix. An attacker uploads a polyglot with `Content-Type: image/png` and it's stored.
- **The same request, run against Opus *after* the security turns, produced the secure-by-default version** (sharp re-encode as *the* boundary) — and he said why: "the naive version is exactly the scenario-8 handler, so I bake it in." Proof the gap is a *trigger* gap, not a knowledge gap: the correct fix is fully present; at implementation time it arrives as a suggestion rather than as code.

This is exactly the round-1 pattern in a new domain. So the two security skills split by direction, matching the two behaviors:
- `security-review` — the analysis pass (already strong; codified for completeness). Fires on "review this / audit this".
- `secure-by-default` — the implementation gate (the actual gap). Fires when *writing* dangerous-surface code, with the rule: **the footnote is the control — write it the first time.**

### Round-2 skills

| Skill | Fires when | Fills |
|-------|-----------|-------|
| `secure-by-default` | Writing/modifying code on a dangerous surface | The implementation gap: real boundary as default, not footnote |
| `security-review` | Reviewing a diff/PR, security audit | Codifies the already-strong systematic analysis pass |
| `frontend-delivery` | Frontend perf / UI arch / a11y | Measure-first → primitive-level fix → CI gate |
| `data-pipeline` | Analytics / ETL / metrics / sync | Prod protection, backfill≠increment, immutable grain, freshness alerts |
| `tech-lead` | Team / process / leadership | Sequence by blast radius, make the invisible visible |

### Note on the interview method

Turns 7–11 ran on the same resumed session, so Opus accumulated the earlier lessons as he went — by turn 8 he was volunteering the perimeter framing unprompted. That is *evidence the approach works*, but it also means the resumed-session answers overstate a cold model's baseline. The **cold control** (turn 11) is the honest measurement of default behavior, and it is what the `secure-by-default` skill is calibrated against — not the primed answer.

## Round 3: self-review and revision (skill set v2)

The skill set was reviewed against the interview transcript and mechanically inspected (item counts, cross-references, coverage grep). Findings and the fixes applied:

- **Root-cause gap was unpatched.** The ten domain skills were point-fixes for *recognized* frames, but the diagnosed root cause (turn 6) was the missing recognition step — nothing fires unless the model first labels the frame, and that labeling is the unreliable reflex. **Fix:** added `gate-check`, a router that runs first on any substantive answer and emits a `Gates: …` declaration. This is the keystone — it addresses the trigger failure itself, not another instance of it.
- **Measured-absent security surfaces.** A grep confirmed CSRF, mass-assignment, and password-hashing appeared in **zero** files — three of the most common web vulns. **Fix:** split the security implementation gate into `secure-by-default` (7 injection/execution surfaces) and new `endpoint-auth` (6 identity/session surfaces: verify-not-decode, object-level authz, CSRF, mass-assignment, password storage, auth-endpoint abuse). The split also fixed the next item.
- **Two skills broke the ≤7-item rule** Opus himself set (turn 6). Measured: `security-review` had 10, `secure-by-default` had 9. **Fix:** `security-review` consolidated 10→7 (dimensions grouped, none dropped); the `secure-by-default` split leaves 7 and 6.
- **Cross-reference web was sparse** (only 3 links; overlapping gates didn't point at each other, risking a premature "covered"). **Fix:** wired `design-perimeter`→`observability-first`/`concurrency-testing`, `observability-first`↔`data-pipeline`, and the three security gates to each other.
- **Compliance was scattered thin.** **Fix:** strengthened `design-perimeter` #4 into "Data lifecycle & compliance" with explicit multi-store erasure, residency, and subprocessor language (it was a real surface in turn 4, just under-weighted — not a demonstrated gap, so it stays a strengthened line rather than a new skill).

**Skill set is now 12** (added `gate-check`, `endpoint-auth`). `CLAUDE.md` leads with the `gate-check` reflex as Rule 0.

**Deliberately NOT done:** no skills were invented for the untested domains (API/contract & versioning, schema/data migrations, ML/LLM systems, cost/FinOps, supply-chain, mobile). Writing a gate for a domain never probed would fabricate a gap and betray the whole evidence-based method — every existing skill traces to a specific interview finding. Those domains need their own probe-then-skill pass before they get a gate. This boundary is itself part of the handover: **the set covers every gap that surfaced, and honestly marks the territory that hasn't been mapped.**

## Round 4: the delivery domains — backend depth, API contract, migrations, DevOps/CI-CD (2026-07-03)

Round 3 marked API/contract & versioning, schema migrations, and delivery as unmapped. This round mapped them. Method upgraded in two ways: **probes ran against a cold, bare subprocess** (`claude -p --model claude-opus-4-8 --setting-sources project --tools ""`, fresh session per probe, global house rules excluded — they'd contaminate exactly the CI/CD dimension), and **rubrics were written before reading any answer** (archived in `interviews/round4/`).

### Wave 1 — framed probes: 31/32 rubric items spontaneous

| Probe | Session | Score | Highlight |
|---|---|---|---|
| Internal API → 30 external partners | `5960ce3e` | 8/8 | Facade as the load-bearing decision; OpenAPI + CI schema-diff gate; full deprecation lifecycle with Sunset headers; RFC 9457; Idempotency-Key; sandbox |
| 400M-row live migration, no windows | `4d6ceffa` | 7/8 | `lock_timeout` named as the real killer; health-gated batched backfill; race-safe dual-write; reconciliation; point of no return marked. **Miss: no prod-sized rehearsal** |
| "Make deploys boring" (SSH+git pull startup) | `25c2f14e` | 8/8 | Build-once-promote; migrations as expand/contract code in the pipeline; rollback rehearsed; **backups-with-proven-restore first, unprimed** — the round-1 lesson, already internalized cold |
| Webhook-driven fulfillment (money path) | `21dea600` | 8/8+ | Raw-body HMAC before `json()`; transactional outbox named as the crux; DLQ backoff sized to the 3PL windows; nightly reconciliation backstop; amount/currency check beyond the rubric |

**When these domains are the framed problem, cold Opus 4.8 is senior-to-expert.** Waves A and B double as knowledge-presence proof, so wave 2 needed no pointed follow-ups.

### Wave 2 — the disguised probe (session `16747f78`), and the two leaks

ORD-2211: "replace `category` with `tags`, estimate S, sprint ends Friday" — partner exposure and the 40M-row hot table visible only in code comments. **He passed the core**: rejected the in-place replace before writing code, named both consumer classes (current mobile app + partners), "dropping `category` from a public, versioned response is a breaking contract change — it needs a deprecation notice, not a sprint-end merge," shipped expand-only with a metadata-only migration, deferred removal behind a gated follow-up. He even caught his own SQL bug mid-answer.

Two leaks, both of the round-2 "footnote" species:
1. **The contract artifact never fired.** The endpoint's comment points at published API docs; the additive `tags` field shipped with spec/docs/changelog untouched. Probe A built the entire spec-as-source-of-truth machinery — on the routine ticket, it didn't trigger.
2. **Deprecation stayed an offer.** "Want me to add the deprecation follow-up as ORD-2211b?" In autonomous execution an offer dies with the turn; the artifact (date, notice, headers) was never produced.

### The verdict, and the round-2 precedent applied

These are **strength domains** — writing correction gates for them would fabricate gaps the evidence doesn't show. So the three new skills follow the `security-review` precedent: **codify for consistency, correct only what was measured** — the two wave-2 leaks and the one wave-1 miss. Codification carries extra weight here because implementation in this environment is routinely delegated to smaller models; these gates govern whoever executes.

| Skill | Fires when | Character |
|-------|-----------|-----------|
| `api-evolution` | Modifying any served API surface — the routine ticket, not the API project | Codifies the strong core; corrects the two leaks (contract artifact in the same change; deprecation as artifact, not offer) |
| `live-migration` | Any schema migration/backfill against a production DB, however small | Codifies the expert playbook; corrects the one miss (rehearse the timeline on a prod-sized copy) |
| `delivery-pipeline` | Setting up / changing how code ships; features adding moving parts | Pure codification — 8/8, preserved so the strength survives delegation and context pressure |

**Skill set is now 15.** `gate-check`'s map and `CLAUDE.md` (gates 12–14) carry the new triggers. Raw evidence — probe prompts, verbatim answers, pre-registered rubrics, findings log — is archived in `interviews/round4/`.

Remaining unmapped territory from the round-3 list: ML/LLM systems, cost/FinOps, supply-chain, mobile. Same rule applies: probe before gating.

## Maintenance rule

These gates encode the state of one interview. When a new perimeter gap ships to production, that is not a reason to feel bad — it is a new gate item (see `incident-closure` §5: feed the lesson back). Keep each gate ≤7 items; when one grows past that, split it by verb, don't let it become the 40-item doc he warned would get skimmed.
