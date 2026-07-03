---
name: concurrency-testing
description: Use when designing, building, reviewing, or modifying ANY system with concurrent state — sync/merge engines, CRDTs, queues, caches, snapshot/persistence paths, multi-writer anything, retry logic, distributed state. Unit tests check states; these systems fail on SCHEDULES. Fires at design time, not after the first irreproducible bug report.
---

# Interleaving Is an Input, Not an Accident

A system whose bugs only appear under interleaved operations, disconnects, and retries must have interleaving as a *controlled input*. If concurrency is an environmental accident in your tests, your first real concurrency bug will be an irreproducible production data-loss ticket. The testing strategy is part of the design — write it in the design doc, not after launch.

## The strategy, in value order

1. **Property-based convergence tests.** State the core invariant explicitly (for CRDTs: replicas that have seen the same set of ops converge to identical state regardless of order; more generally: no acknowledged write is ever lost). Generate random op sequences, apply in random interleavings, assert the invariant. Highest value per line of test code.

2. **Deterministic simulation testing** (the FoundationDB approach). Simulated network and clock; message ordering, delay, duplication, reordering, partition, disconnect/reconnect all driven by a **seeded RNG**. Run large numbers of randomized schedules in CI; any failure replays *exactly* from its seed. This converts "nobody can reproduce it" into "replay seed 4187". Requires designing for it — inject the clock and transport from day one; it cannot be bolted on later.

3. **Fault injection at the seams you designed.** Every architectural seam you created is a scheduled adversary: cache eviction mid-write, the un-persisted tail between snapshot and crash, the disconnect during flush. Script each one as an explicit adversarial schedule. If you can name the seam in the design review, you can write its attack before it ships.

4. **Durability checking on the persistence path** (Jepsen-style): under partition and process kill, assert no acknowledged edit is lost. "Acknowledged" is the word that matters — define exactly when the system says "saved" and test that promise, not the happy path.

5. **Regression corpus.** Every real incident becomes a permanent seeded test that replays forever (see `incident-closure` §2). The corpus is the system's institutional memory.

## The gate

Before calling a concurrent-state design done, write one sentence for each: *what is the invariant*, *how is the schedule controlled in tests*, *which seams get adversarial schedules*, *what does "acknowledged" mean*. Blanks mean the testing strategy doesn't exist yet.

## Why this exists

Handover-review finding: four consecutive design scenarios on a concurrency-critical system without the word "test" appearing once — followed, when prompted, by an expert-grade answer. The knowledge is present; the trigger is not. This skill is the trigger.
