---
name: tech-choice-gate
description: Use when evaluating a rewrite, a new language, a framework swap, a build-vs-buy call, or any "we should use X" / "Y is too slow" proposal — before agreeing, estimating, or designing. Performance claims are hypotheses; team topology is part of the tradeoff.
---

# Tech Choices Are Evidence Decisions

"X is too slow" and "we should use Y" are hypotheses wearing conclusions' clothes. Run this gate before any answer that endorses, rejects, or estimates a technology change. Each item gets a written answer.

1. **Demand the evidence first.** For performance claims: flame graphs and attribution *at the tail* (p99/p999), showing the proposed fix targets the *dominant term*. The tail is usually I/O, round-trips, and serialization — not the CPU work people want to rewrite. No profile, no rewrite.

2. **Exhaust the cheaper fixes in the current stack — list them.** Worker threads, pooling, moving work off the hot path, caching, algorithmic fixes. Each rejected cheap fix needs a reason; "we already decided" is not one.

3. **Swap a library, don't rewrite an app.** Look for the bounded adoption first (e.g. adopting a proven native core behind the existing wire protocol vs. greenfielding the engine). The blast radius of a dependency swap is a fraction of a rewrite's.

4. **Price the polyglot tax and the bus factor.** Two toolchains, two CI paths, two debugging stories, an FFI boundary — forever. If 2 of 6 engineers know the new language, the most safety-critical code becomes maintainable by a third of the team. That is an organizational risk, not a style preference. Say it out loud in the recommendation.

5. **A rewrite discards baked-in fixes.** Every hard-won bug fix in the old implementation is silently deleted by a rewrite. Price the re-discovery of those bugs into the estimate.

6. **Buy-vs-build: is this our differentiator?** If the capability isn't what customers choose the product for, default to buying/adopting — and for anything touching customer data, treat the vendor as a subprocessor: residency, DPA, and exit path are part of the evaluation, not an afterthought.

7. **Defer with a trigger, don't dismiss.** When the answer is "not now", write the concrete condition that reopens the decision ("when profiling shows merge CPU > 40% of tail", "when team has 4+ people fluent"). Separate the *outcomes* the proposer wants from the *tool* they named — concede the outcomes, negotiate the tool.

## Why this exists

Handover-review finding: this judgment was consistently present when asked directly — the gate exists to make sure it's applied *before* enthusiasm or authority (a CTO's "this is embarrassing", a teammate's "GC pauses hurt us") frames the decision as already made.
