---
name: setup-claude-expert-mode
description: Install and configure Claude Expert Mode (the verb-triggered perimeter-gate skillset) in a target project or globally. Toggle which skills to install, wires matching House Rules into CLAUDE.md, trims the gate-check router map, records a manifest so re-running updates or prunes a previous install.
disable-model-invocation: true
---

# Setup Claude Expert Mode

Claude Expert Mode is a family of **verb-triggered perimeter gates** produced by an evidence-based knowledge handover (see `HANDOVER.md` at the pack root). Each skill encodes a measured finding: either a demonstrated gap (correction) or a demonstrated strength preserved for consistency (codification). They share three conventions: gates fire on verbs, every gate demands written output, and an omission is always visible ("N/A because X", never blank).

This is a prompt-driven skill, not a deterministic script. Explore, present, let the user toggle, confirm, then write.

## The catalog

**Source of truth:** the sibling directories of this skill's own folder (`../<name>/SKILL.md`) plus the House Rules in the `CLAUDE.md` two levels up (the pack root). Always read the source at install time — never install from this table alone; it's the menu, not the food.

| Group | Skill | One-liner |
|---|---|---|
| **Core** | `gate-check` | The router — emits `Gates: …` before any substantive answer. The keystone; everything else assumes it. |
| **Design & decisions** | `design-perimeter` | Six-line operational perimeter appended to every finished design/plan |
| | `tech-choice-gate` | Rewrite / new-tool / buy-vs-build proposals: profile first, price the tax, defer with trigger |
| | `tech-lead` | Team/process/leadership: sequence by blast radius, make the invisible visible |
| **Security** | `secure-by-default` | Untrusted input → dangerous sink: the real boundary is written first time, never a footnote |
| | `endpoint-auth` | Identity/session surfaces: verify-not-decode, object authz, CSRF, mass-assignment, passwords |
| | `security-review` | Review-time systematic pass: trace every input to every sink, report everything |
| **Build & data** | `concurrency-testing` | Concurrent state at design time: invariant stated, interleaving a seeded input |
| | `data-pipeline` | Analytics/ETL/sync: off-prod, backfill≠increment, immutable grain, freshness alerts |
| | `frontend-delivery` | Frontend perf/a11y: measure the user metric first, fix at primitive level, CI-gate it |
| **Ops & incidents** | `observability-first` | Any infra plan: restore-test today, RPO/RTO as numbers, day-one alert kit |
| | `incident-closure` | Root cause is the midpoint: prove THE bug, recover victims, close, retro |
| **API & delivery** | `api-evolution` | Modifying a served API surface: consumers inventoried, additive/breaking classified, contract artifact updated in the same change, deprecation as artifact not offer |
| | `live-migration` | Any production schema migration: expand/contract, lock class + `lock_timeout`, health-gated backfills, N/N+1 compatible, rehearsed |
| | `delivery-pipeline` | How code ships: build-once-promote, CI merge gate, rehearsed rollback, migrations decoupled |

**Cross-references** (used for dependency warnings in step 3):

- `gate-check` → all installed skills (its map routes to them)
- `design-perimeter` → `observability-first`, `concurrency-testing`
- `secure-by-default` ↔ `endpoint-auth` ↔ `security-review`
- `observability-first` ↔ `data-pipeline`
- `api-evolution` → `design-perimeter`, `live-migration`
- `live-migration` → `delivery-pipeline`, `api-evolution`, `data-pipeline`
- `delivery-pipeline` → `live-migration`, `observability-first`

**House Rules mapping** (pack-root `CLAUDE.md` → skill): Rule 0 = `gate-check`; 1 = `design-perimeter`; 2 = `incident-closure`; 3 = `observability-first`; 4 = `concurrency-testing`; 5 = `tech-choice-gate`; 6 = `secure-by-default`; 7 = `endpoint-auth`; 8 = `security-review`; 9 = `frontend-delivery`; 10 = `data-pipeline`; 11 = `tech-lead`; 12 = `api-evolution`; 13 = `live-migration`; 14 = `delivery-pipeline`. The meta-rule ("N/A because X") always ships.

## Process

### 1. Explore

- Resolve the **pack root**: this skill's directory's parent is the skills dir; two levels up is the pack root (with `CLAUDE.md` and `HANDOVER.md`). If run from a copy that lost its siblings, stop and ask for the pack path.
- Look at the **target**: is this a project repo (has `.git`, or the user is working in one) or does the user want a global install? The global root is `$CLAUDE_CONFIG_DIR` when that env var is set, `~/.claude` otherwise — resolve it, don't assume; multi-account setups run several config dirs. If the global `skills/` dir is a symlink (shared-folder setups), install through it — never replace the symlink with a real directory.
- Check for a previous install: `expert-mode-manifest.md` next to the target's skills dir. If present, this run is an **update/sync**, not a fresh install — read it and diff against the new selection in step 3.
- Check the target's `CLAUDE.md` / `AGENTS.md` for an existing `## House Rules — Operating Perimeter` (or `## Claude Expert Mode`) block. Follow `@path` import lines while looking: the block may live in an imported file (conventionally `gate-checks.md`; older installs used `house-rules.md`) — the imported file is then the edit target in step 4, not the importing `CLAUDE.md`.
- Check the target's skills dir for name collisions with skills that did NOT come from this pack (not listed in a manifest) — never silently overwrite those; surface them.

### 2. Ask scope and toggles

Walk through **one decision at a time**; assume the user hasn't read the handover.

**Section A — Where to install.**

> Explainer: project install (`<repo>/.claude/skills/`) gates only that repo and travels with it in git; global install (`<config-dir>/skills/`, where the config dir is `$CLAUDE_CONFIG_DIR` or `~/.claude`) gates every session on this machine. The House Rules block lands in a `gate-checks.md` imported from the matching CLAUDE.md (project, or the config dir's — or whatever imported file already holds the block, per step 1 and step 4.2).

**Section B — What to install.** Present the six groups with their one-liners. Offer three modes:

- **Full set** (recommended) — all 15. The set was designed as a web; cross-references stay intact.
- **By group** — toggle the six groups (multi-select). `Core` (`gate-check`) is strongly recommended whatever else is picked: it is the trigger reflex the whole handover exists for; without it the domain gates only fire when someone thinks of them — which is the measured failure mode.
- **Individual** — per-skill toggle for users who know exactly what they want.

### 3. Resolve dependencies and confirm

- For every selected skill, check its cross-references. If a referenced sibling is not selected, tell the user: "`live-migration` points at `delivery-pipeline` for how migrations ship — install it too, or accept a dangling mention?" Offer to add the sibling (default) or proceed (the mention becomes aspirational text, not an error).
- If this is an update/sync: show the diff — skills to add, skills to update in place, previously-installed skills now deselected (offer to remove them and their House Rules lines; never remove non-manifest skills).
- Show the final plan: skill list, target paths, and a draft of the House Rules block (step 4). Let the user edit before writing.

### 4. Write

1. **Copy skill directories** from the pack to the target skills dir. Copy, don't symlink — the target may be committed to git or leave this machine. Never copy `setup-claude-expert-mode` itself into a project (it stays in the pack / global skills).
2. **House Rules block.** Where it goes, in order of preference:
   - **Existing block behind an `@` import** (step 1): that imported file is the edit target — never also write into the importing `CLAUDE.md` (that duplicates it).
   - **Existing inline block**: replace it in place (or offer to extract it to `gate-checks.md` + an `@` import — recommended, see next).
   - **Fresh install (no block anywhere)**: create **`gate-checks.md`** next to the target `CLAUDE.md`/`AGENTS.md`, write the block there, and append a single import line (`@gate-checks.md`, or the relative path) to `CLAUDE.md` if it exists, else `AGENTS.md`, else ask which to create. The curated file stays hand-edited; the machine-replaceable block lives in its own file. Write the block inline only as a fallback when the target can't follow imports.

   Build the block from the pack-root `CLAUDE.md`:
   - Take the intro paragraph, Rule 0 (only if `gate-check` is installed), the numbered rules whose skills are installed (renumber sequentially so there are no holes), and the meta-rule.
   - If a previous Expert Mode block exists, replace it in place; don't duplicate; don't touch surrounding user content.
3. **Trim `gate-check`'s map** in the *installed copy* (never in the pack): delete map rows pointing at skills that aren't installed. If a full-set install, no trim. Leave the three rules and everything else untouched.
4. **Write the manifest** — `expert-mode-manifest.md` next to the installed skills:
   - install date, pack source path, mode (full/group/individual)
   - the installed skill list
   - any accepted dangling references
   - one line: "Managed by `setup-claude-expert-mode` — re-run it to update or change the selection."

### 5. Done

Tell the user: which gates are now active, that they fire on verbs (nothing to invoke manually — `gate-check` routes), and that the evidence behind every gate lives in the pack's `HANDOVER.md` and `interviews/`. Re-run this skill to add/remove skills or pull updated gate text after a new handover round; hand-edits to installed copies survive until the next sync overwrites them (warn about that at sync time by diffing before overwrite).
