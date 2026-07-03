# Interview evidence

Raw evidence behind every gate. See `../HANDOVER.md` for the findings each round produced.

- **`rounds1-2/`** — the original knowledge-transfer interview (one resumed session, 22 turns): round 1 scenarios (architecture, debugging, infrastructure, security, tech choice + testing) and round 2 (frontend, data engineering, code security both directions, team/process), plus the primed implementation turn. Includes the **cold control** (separate unprimed session, avatar-upload endpoint) that `secure-by-default` is calibrated against.
- **`round3/`** — the skill-set self-review and revision (v2): the review that found the unpatched root cause and produced `gate-check`, the `endpoint-auth` split, the ≤7-item enforcement, and the cross-reference web. A working-session transcript, not an Opus interview.
- **`round4/`** — the delivery domains (API contract, live migrations, DevOps/CI-CD, async backend), run with the upgraded method: cold bare subprocess per probe, pre-registered rubrics (`rubrics-round4.md`, written before reading any answer), five probes (four framed + one disguised), findings log, and the verbatim cold answers.
- **`verification/`** — the before/after proofs: the same prompts re-run against Opus 4.8 **with the gates installed**. Compare `verification/avatar-upload-gated.md` with `rounds1-2/cold-control-avatar-upload.md`, and `verification/round4-disguised-ticket-gated.md` with `round4/answer-E.md` — same model, same ask; the delta is the pack.
