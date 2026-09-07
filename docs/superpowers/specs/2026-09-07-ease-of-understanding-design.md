# Ease of understanding: docs, comments, simplification

Working document for the pass that makes the umbrella, firmware, backend and app repositories
readable by a newcomer. Delete it when the pass is done. `cad` and `plan` are out of scope.

## Goal

Two readers. A new developer finds where to edit or add code in under ten minutes. A new user
builds, configures and runs each part from the README alone. Neither needs the plan, the
decisions log, a past pull request or a session transcript.

## Writing rules

1. Delete every comment that is not useful. Shorten the useful ones to the essentials.
2. Short docs people read beat long ones nobody opens.
3. No jargon or acronyms unless they earn their place, then define on first use in every document
   that uses them. `GLOSSARY.md` in the umbrella is the reference, not a substitute for that.

Comment policy, copied into every `AGENTS.md`:

- A file starts with one line saying what it holds.
- A comment says why, not what. Keep invariants, units, hardware traps, security reasons.
- No dates, task numbers, PR numbers, pitch names, decision numbers, reviewer names, or history
  ("used to", "since 0.17.0"). Keep the fact, drop the provenance.

## What gets written

Umbrella:

- `README.md`: start here. One diagram of board, backend and app and what travels between them.
  A user path (deploy backend, flash board, install app) and a developer path (pick a repo).
  Clone commands that work without private tools.
- `GLOSSARY.md`: one line per term for the whole system: pot, hose, controller, channel, outlet,
  dose, band, mapping window, contra, latch, cart, gate, drop, NAS, tailnet, `k=v`, GBIF,
  Trefle, ntfy, and the alert key prefixes.
- `DECISIONS.md`: an index table at the top (number, title, gist, superseded by). Entries stay
  untouched, per its own rule.
- `AGENTS.md`: agent-only, under 60 lines. "Where we are" keeps its newest entry; older entries
  are deleted (git history has them).
- Org profile README in `plantbutler/.github`: fix the stale "rotary manifold", point at the
  umbrella README.

Each code repo:

- `README.md`, under 80 lines: what it is in three sentences, how it fits (link to the umbrella),
  build, test, run and deploy as commands, a configuration table, a file map with one line per
  file, a link to the glossary.
- `AGENTS.md`, under 60 lines: rules, checks, traps, where truth lives, the comment policy.

## Decisions taken (2026-09-07)

1. Umbrella diary: newest entry stays, the rest is deleted.
2. `docs/superpowers/` in firmware, backend and app is deleted once the README carries what it
   was the only source of (firmware module map, app test command, wire contract for refill and
   resume).
3. `butler.py` is split into modules as part of the backend simplification step.
4. Renames only where a name lies: `waters` to `may_water`, `free_alerts` to `clear_alerts`,
   `slurp` to `read_body`, `keep_photo` and `forget_photo` to `store_photo` and `delete_photo`,
   the app's `flight` and `strip`. Domain words (contra, cart, latch) stay and go in the glossary.
5. Docs live in the repositories as markdown. No wiki.
6. Simplification of code and tests is part of this pass, as its own step per repo.

## Order

Umbrella first; its docs do not depend on code shape. Then firmware, backend, app. Inside each
code repo, four steps, each its own PR:

1. Comment cleanup under the policy. Delete stock PlatformIO READMEs, `constants.h`, and
   `docs/superpowers/` once absorbed. Tests unchanged and green.
2. Simplify code with tests untouched. Opens with a survey that lists candidates and their
   size before any edit. Behaviour preserved; the dry-failure invariants in decisions 5 and 7
   survive. Includes light structure: section banners, the alert prefix list as one constant,
   a `conftest.py`, the `butler.py` split, region markers in `GardenViewModel.kt`.
3. Simplify tests with code untouched. Same survey first.
4. `README.md` and `AGENTS.md`, written against the final shape.

## Verification

- Tests green after every step; firmware also `make check`.
- Newcomer test per repo at the end: a fresh agent given only the repo's README, AGENTS.md and
  the glossary must answer how to build, test, run, and where to add an endpoint, a screen, or a
  serial command. Docs are not done until it passes.
- Every deletion reviewed as a diff before commit.

## Traps

- Firmware `tools/check.sh` greps comment wording and file include lists; a comment edit can
  fail the build. Run `make check` after every firmware edit. It exits 2 on a clean clone by
  design; the README must say so.
- Firmware `platformio.ini` `build_src_filter` replaces rather than extends, so which files
  compile depends on the environment. The README must say so.
- Backend: a column exists only if it is in both `schema.sql` and `ADDED_COLUMNS`.
- Other sessions edit these repos. One worktree per repo for this pass, branch `readable`,
  one PR per step, rebased before merge.
- `DECISIONS.md` entries are never edited. The index is additive.
- Secrets never enter a repository; the README points at the sample files.
