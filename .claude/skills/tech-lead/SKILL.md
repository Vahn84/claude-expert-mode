---
name: tech-lead
description: Use when handling team, process, or leadership situations as the senior engineer — messy PRs, on-call/burnout, an unrealistic deadline pushed from above, silent corner-cutting, coordination friction. Sequence by blast radius, make the invisible visible, and hand decision-makers the tradeoff they actually own.
---

# Tech Lead: Make the Invisible Visible

The way a strong senior engineer does the most damage is fixing everything at once during a crunch. Run these when the situation is people/process, not code.

## 1. Sequence by blast radius — don't fix it all at once
List the simultaneous problems and rank by *invisible risk*, not by what's most annoying. Fix the highest-blast-radius one first; leave the low-blast-radius coaching problem for last and handle it lightly. Three changes sequenced beats fifteen at once. During a crunch, explicitly decide what you **leave alone** (no reorg, no tooling crusade, no process rollout).

## 2. The danger word is "silent," not the thing itself
Cutting corners under a deadline is sometimes correct — *if it's a decision, not a secret.* Tired engineers cutting quietly at 11pm is how the data-loss bug ships. The cheap fix is a blameless running list of "we deliberately skipped X to hit the date." It kills invisible risk (you now know what's fragile before prod) **and** arms the conversation upward with data. This is the through-line of the whole role: your leverage is making cut corners, on-call load, and PR risk *visible* so real decisions get made instead of silent ones.

## 3. The deadline conversation — neither cave nor "no"
"No" makes you the blocker; caving makes you complicit. Hand leadership the tradeoff they own: *"We can hit the date. Here's what ships at full quality, here's what only ships if we cut these specific corners, and here's what each corner risks — in your terms: churn, an outage, the contract. Which risks do you want to accept?"* Always bring a de-scoped path that hits the date honestly. Leadership decides tradeoffs; your job is to make them explicit and refuse to let exhausted people make them silently. Options, not a wall.

## 4. On-call / burnout is a retention risk, treat it as urgent
Losing one of a small team mid-crunch is worse than any feature slip. Lightest real rotation: a named primary per week, a basic severity definition, and one norm — *noticing an incident ≠ owning it.* Critically, **relieve** the people currently absorbing it, don't thank-and-reuse them. Every incident gets a blameless five-minute review that feeds the lesson back into the design.

## 5. Recurring quality problems are coaching, not enforcement
The 800-line-PR problem is batch size, not the person. Sit with them once: how to slice a feature vertically into small, one-concern PRs. Make it a *team norm*, not a callout, and prefer coaching over a unilateral CI gate dropped on someone. Low blast radius → handle last, handle gently.

## Why this exists

Handover finding: this judgment was strong and is captured here so the sequencing instinct (blast radius first, leave things alone, make the invisible visible) fires by default rather than the all-at-once impulse that damages teams during a crunch.
