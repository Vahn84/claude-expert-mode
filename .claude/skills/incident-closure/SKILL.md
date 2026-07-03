---
name: incident-closure
description: Use when a bug's root cause has been identified, a fix has been written, or you are wrapping up any incident, data-loss report, outage, or production-issue investigation. Finding the mechanism is the MIDPOINT of an incident, not the end — this skill defines the second half.
---

# Mechanism Found ≠ Incident Closed

The satisfying moment — "found it, here's the race" — is where investigations get abandoned. It is the halfway point. Do not report an incident as resolved until every item below has a written answer.

## 1. Confirm it's THE bug, not A bug
The mechanism must *quantitatively* explain the pattern: the frequency (~why 5/week and not 500), the specificity ("last few minutes" and not random chunks), the silence (why logs showed nothing), and the irreproducibility. A plausible mechanism that doesn't explain the pattern is *a* bug — fix it and keep looking.

## 2. Regression test that replays the exact failure
Not a test of the general area — a test that reproduces *this* loss and fails without the fix. For concurrency bugs: a seeded, deterministic schedule (see `concurrency-testing`), so "can't reproduce" becomes "replay seed 4187".

## 3. Recover the victims
The fix protects future users. What about the ones already hit? Check whether the data is recoverable (update logs, WALs, backups, client-side copies) and backfill it. If it is not recoverable, that is a fact to communicate, not to skip.

## 4. Close the loop with humans
Every support ticket that triggered the investigation gets an answer: what happened, what was recovered, what changed. Silence after "we're looking into it" is how a bug becomes a churned customer.

## 5. Feed the lesson back
Run the retro question: *which design assumption did this break?* ("on-idle snapshots are sufficient" → they weren't). Update the design doc, the runbook, or the relevant skill so the class of bug — not just the instance — is covered. If the incident revealed a missing gate, add the gate.

## 6. Verify closure over time
Reports go to zero **and stay zero for a full cycle** (whatever a cycle is for the observed frequency — a 5/week bug needs multiple quiet weeks). Add a metric/alert for recurrence so regression is a page, not a slow trickle of tickets.

## Why this exists

Handover-review finding: root-cause analysis was excellent, and then the incident was mentally closed at "mechanism found" — no backfill, no ticket closure, no retro, until prompted. The second half of an incident is unglamorous and has no intellectual pull. Run the list anyway.
