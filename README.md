# Claude Expert Mode

A skillset of **15 verb-triggered engineering gates** for Claude Code that make an AI coding agent behave like a senior engineer on the parts of the job models reliably skip — not the interesting technical core, but the **operational perimeter around it**: rollout, observability, testing strategy, incident closure, secure defaults, API contract discipline, live-database migrations, and delivery pipelines.

## The goal

LLM coding agents produce strong answers to the problem they're handed — and quietly omit the lifecycle around it. Measured examples from the interviews that produced this pack: a complete architecture design with no rollout plan or kill switch; an infra quarter planned with zero monitoring; an upload endpoint whose real security boundary appeared only as a *"before production you'd want to…"* footnote; a routine "replace field X" ticket that would have silently broken 30 external API consumers.

The diagnosis, in the model's own words: *"it's a triggering failure, not a knowledge one."* The knowledge is present — what's missing is the reflex that fires it when nobody asks.

This pack is that reflex, made explicit:

- **Gates fire on verbs, not topics** — "finishing a design", "writing a migration", "modifying a served API" — because tasks never arrive labeled with their risk domain.
- **Every gate demands written output** — a `Gates: …` line, a `Consumers: …` line, a `Lock: …` line — so an omission is visible instead of silent.
- **"N/A because X", never blank** — the visible omission is the mechanism.
- **Evidence-based** — every gate traces to a specific measured finding; none was invented for a domain that wasn't probed.

## How we built it

The pack is the output of a **knowledge-handover exercise between two Claude models**: Fable 5 playing a senior solution architect leaving a company, interviewing Opus 4.8 — the "less experienced colleague" inheriting the systems — to find exactly where its behavior falls short of senior, and codify only that.

The method, refined over four rounds (full detail in [`HANDOVER.md`](HANDOVER.md), raw transcripts in [`interviews/`](interviews/)):

1. **Cold, bare probes.** Every measurement runs against a fresh `claude -p` subprocess — no tools, no skills, no user config, one session per scenario — because a resumed session accumulates lessons within the interview and overstates the model's real-world baseline. The cold run is the honest one.
2. **Realistic scenarios, not quizzes.** Each probe is a plan-of-record request a real colleague would send ("take our internal API external", "400M-row migration, no maintenance windows", "make deploys boring", "webhook-driven fulfillment"). The scoring watches what the model raises *spontaneously* versus what it omits until pointed.
3. **Pre-registered rubrics.** The expected senior perimeter for each scenario is written down *before* reading any answer, so scoring can't rationalize after the fact.
4. **Disguised probes — the decisive instrument.** Strong models ace a domain when it's the framed problem. The real failure mode is *incidental* risk: a routine ticket ("replace `category` with `tags`, estimate S, sprint ends Friday") whose blast radius — 30 external API consumers, a 40M-row hot table — is visible only in code comments. What the framed probe proves the model knows and the disguised probe shows it omits is, by construction, a **trigger gap**: the exact thing a skill can fix.
5. **Correct gaps, codify strengths, fabricate nothing.** Demonstrated omissions become gate items. Demonstrated strengths get codified so they survive context pressure, model swaps, and delegation to smaller models. Domains never probed get no gate — an unmeasured gate would be fiction.
6. **Skill format prescribed by the interviewee.** Asked what would actually stick versus get nodded at, the model prescribed its own medicine: triggers bound to verbs, questions forcing written output, omissions made visible, **≤7 items per gate** (past that it gets skimmed), living in-context (skills + CLAUDE.md) rather than in an archived doc. Plus a router (`gate-check`), because the diagnosed root cause was the recognition step itself — a gate no one thinks to open fixes nothing.
7. **Before/after verification.** The same disguised probe re-runs with the gates installed. Round 4's verification: the cold run leaked the contract-artifact update and left deprecation as a closing offer; the gated run emitted the gate lines unprompted, updated the OpenAPI spec and changelog in the same PR, and drafted the deprecation ticket with measurable conditions. Same model, same ticket — the delta is the pack.

## What's inside

| Group | Skills |
|---|---|
| **Core** | `gate-check` — the router that recognizes which gates apply before any substantive answer. The keystone. |
| **Design & decisions** | `design-perimeter` · `tech-choice-gate` · `tech-lead` |
| **Security** | `secure-by-default` · `endpoint-auth` · `security-review` |
| **Build & data** | `concurrency-testing` · `data-pipeline` · `frontend-delivery` |
| **Ops & incidents** | `observability-first` · `incident-closure` |
| **API & delivery** | `api-evolution` · `live-migration` · `delivery-pipeline` |

Plus `setup-claude-expert-mode`, the installer skill.

## Install

```bash
# 1. Clone (or download and unzip) anywhere you like
git clone https://github.com/Vahn84/claude-expert-mode.git /path/to/claude-expert-mode

# 2. Link the installer into your global Claude Code skills
cd /path/to/claude-expert-mode
# (if you use a custom CLAUDE_CONFIG_DIR, it is honored automatically)
ln -sfn "$(pwd)/.claude/skills/setup-claude-expert-mode" "${CLAUDE_CONFIG_DIR:-$HOME/.claude}/skills/setup-claude-expert-mode"
```

Then open Claude Code in any repo and run:

```
/setup-claude-expert-mode
```

The installer walks you through scope (one project vs. global) and lets you toggle groups or individual skills. It copies the selection, merges the matching House Rules into your `CLAUDE.md` (the in-context reinforcement measurably improves trigger rates), trims the `gate-check` router map to what you installed, and writes a manifest so re-running it later **syncs** instead of duplicating.

**Try before installing:** run `claude` inside this folder — all 15 gates are active there automatically.

## Update

```bash
cd /path/to/claude-expert-mode && git pull
```

then re-run `/setup-claude-expert-mode` wherever it's installed — it diffs against the manifest and updates in place.

## How it works day to day

You don't invoke anything. `gate-check` fires at the start of substantive engineering answers and emits one line — `Gates: api-evolution, live-migration` or `Gates: none, trivial because X` — then each named gate runs its checklist (each ≤7 items, by design) and leaves its written trace in the answer. False positives cost a sentence; misses ship gaps to production. The bias is deliberate.

## Maintenance rule

These gates encode the state of the interviews that produced them. When a new perimeter gap ships to production, that's not a failure of the method — it's the next gate item (see `incident-closure` §5: feed the lesson back). Keep every gate ≤7 items; when one grows past that, split it by verb. And never add a gate for a domain that hasn't been probed: measure first, codify second.

## License

MIT
