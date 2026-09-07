# Readable backend: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A newcomer can run the backend, find where to add an endpoint, a column, a rule or an alert, and read a comment that tells them something the code does not.

**Architecture:** Four PRs on branch `readable` in `.worktrees/readable-backend`, in order: comments; code simplification including the split of `butler.py`; test simplification; README and AGENTS.md. Each merges before the next starts.

**Tech Stack:** Python 3, `uv`, pytest, SQLite, one Docker container.

**Spec:** `docs/superpowers/specs/2026-09-07-ease-of-understanding-design.md` (umbrella).

## Global constraints

- `uv run pytest` green after every task. The suite is the only guard the backend has.
- Behaviour on the wire never changes: the `k=v` report and response, every route's status codes and body, every alert key.
- A column exists only if it is in both `schema.sql` and `ADDED_COLUMNS`.
- The dry-failure invariants hold: every watering gate refuses rather than waters.
- Secrets and the NAS address never enter the repository.

---

### Task 1: Comment cleanup (PR 1)

Policy per the spec: a file starts with one line saying what it holds; a comment says why, not what; no dates, versions, decision numbers, pitch names, reviewer names or history. Keep units, invariants, security reasons, traps, wire facts.

- [ ] Agent A: `butler.py` (5,540 lines, about 1,300 comment or docstring lines). The module docstring becomes about fifteen lines: what the service is, who talks to it, the failure direction, where the rest is written down.
- [ ] Agent B: `tests/` (17 files, about 10,000 lines) and `fake_device.py`.
- [ ] Agent C: `schema.sql`, `Dockerfile`, `pyproject.toml`.
- [ ] `uv run pytest`. Reviewer over the diff: any deleted comment carrying an invariant, a unit or a security reason comes back, shortened.
- [ ] Commit per agent, PR.

### Task 2: Simplify code, including the split (PR 2)

- [ ] Survey first: duplication, long functions, dead code, names that lie.
- [ ] Split `butler.py`. Its top-level 2,372 lines are pure functions; `create_app()` holds the remaining 3,167 with 46 nested definitions. Modules: the wire parsers, the schema and migration, the tank and latches, the species lookup and bands, the pot and photo parsing, the watering rules, the alert rules, the routes. `butler.py` keeps `create_app` and the names the tests reach for.
- [ ] Known trap: tests monkeypatch `butler.new_pot_id`, `butler.write_new_file` and `butler.FLOW_FLOOR_ML_S`. A patch on the `butler` module does not reach a caller in another module, so about six patch targets move with the code. That is the one place this task touches a test, and every such line goes in the PR body.
- [ ] Alert keys: one table of prefix, meaning and whether it pages once, in place of ten string literals spread through a 500-line function.
- [ ] Renames where the name lies: `waters` to `may_water`, `free_alerts` to `clear_alerts`, `slurp` to `read_body`, `keep_photo` and `forget_photo` to `store_photo` and `delete_photo`.
- [ ] `conftest.py` for the fixtures three test files declare separately.
- [ ] Section 6 of the working principles over the diff. Reviewer. PR.

### Task 3: Simplify tests (PR 3)

- [ ] Survey `tests/`: duplicated fixtures and arrangement, cases proving one thing twice, helpers each file redeclares.
- [ ] Apply; code untouched; a case goes only where another proves everything it asserted. Reviewer. PR.

### Task 4: README and AGENTS.md (PR 4)

- [ ] `README.md` under 80 lines: what it is, how it fits, run locally, run the tests, the environment variable table, deploy as literal `docker build` and `docker run` lines, the route table, the file map, a link to the glossary.
- [ ] `AGENTS.md` under 60 lines: the schema rule, the failure direction, the comment policy, the deploy rule, the traps.
- [ ] Delete `docs/superpowers/`.
- [ ] Newcomer test: a fresh agent with the README, AGENTS.md and the glossary answers how to run it, how to test it, how to deploy it, and where to add an endpoint, a column, a rule and an alert.
- [ ] PR.
