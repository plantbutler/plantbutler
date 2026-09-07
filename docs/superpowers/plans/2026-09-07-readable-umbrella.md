# Readable umbrella: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A newcomer opens the umbrella repository and, from its README alone, understands what the system is, how its three parts talk, and where to go next as a user or as a developer.

**Architecture:** Five markdown files, no code. The README is the entry point; the glossary is the one place terms are defined; the decisions log gains an index and loses nothing; the agent guide shrinks to rules.

**Tech Stack:** Markdown, git, `gh`.

**Spec:** `docs/superpowers/specs/2026-09-07-ease-of-understanding-design.md`

## Global constraints

- Writing rules and comment policy from the spec apply to every line written here.
- `README.md` under 120 lines. `GLOSSARY.md` one line per term. `AGENTS.md` under 60 lines.
- `DECISIONS.md` entries are never edited. The index is prepended; nothing below it changes.
- No private paths (`~/projects/openproj`), no tool a newcomer lacks in any command.
- Clone and submodule commands are run once in a scratch directory to prove they work.
- Branch `readable` in `.worktrees/readable`, one PR against `main`. The org profile is a second PR in `plantbutler/.github`.

---

### Task 1: README.md

**Files:** Modify `README.md` (rewrite).

Content, in this order:
- One paragraph: what the system is, for whom, the three parts and where each runs.
- A diagram in a fenced code block: board, backend, app, and what travels on each arrow (the board posts a `k=v` report and gets back an interval and at most one command; the app calls the backend over HTTP with the same token). Below it, four sentences saying the same in words.
- "Get started as a user": numbered steps, one line each, linking the repo READMEs: hardware (`cad`), backend on the NAS (`backend`), firmware on the board (`firmware`), app on the phone (`app`). Each link names the section it expects to find.
- "Get started as a developer": clone with `--recurse-submodules`, the `git submodule update --init --recursive` fallback, one sentence on what a submodule pointer is, a table of the five repos with one line each and their current state (backend 0.20.0 deployed; app on the phone; firmware bench-tested), where the decisions and the glossary are.
- Define on first use: NAS, submodule, `k=v`. Link `GLOSSARY.md` once.

- [ ] Write the file.
- [ ] Run the clone command from the README in the scratchpad against `origin` and confirm all five submodules populate.
- [ ] Commit: `README: a start-here for users and developers`.

### Task 2: GLOSSARY.md

**Files:** Create `GLOSSARY.md`.

Content: alphabetical table `term | meaning`, one line per term, plain English, at most 25 words. Terms: the wiring nouns (pot, hose, controller, channel, outlet, board), the manifold nouns (manifold, cart, gate, meter, float, tank, position), the watering nouns (dose, band, cooldown, daily cap, quiet hours, proposal, approve, verdict, learning and auto mode, soak, slosh), the latches (dry, contradiction, flap, backend latch), the record nouns (mapping window, graveyard, retired board, species lookup, care source, advice), the wire (`k=v`, `X-Token`, report interval, ack), the services (GBIF, Trefle, ntfy, dead-man URL, ticker), the network words (NAS, tailnet), the tools (PlatformIO, uv, adb, APK). Then a second table `alert key | raised when` for every alert prefix the backend raises. Sources: the collected facts table from the audit agent; when it and the code disagree, the code wins.

- [ ] Write the file.
- [ ] Grep each term in the four repos' READMEs and AGENTS.md; every term in the glossary is used somewhere, every jargon term used there is in the glossary.
- [ ] Commit: `GLOSSARY: every term the system uses, one line each`.

### Task 3: DECISIONS.md index

**Files:** Modify `DECISIONS.md` lines 1-5 only (the preamble). Append nothing, edit no entry.

Content: after the preamble, a table `# | date | title | notes` with one row per entry, 31 rows, number 17 absent. Titles verbatim from the entries. Notes only for supersedes / superseded by / partly reversed by / made real by, from the supersession map. Two sentences above the table: it is an index, entries stay as written, the notes are the only place a later reversal is recorded next to the original.

- [ ] Write the index.
- [ ] `git diff DECISIONS.md` shows additions only above the first `## 2026-` heading.
- [ ] Commit: `DECISIONS: an index, entries untouched`.

### Task 4: AGENTS.md

**Files:** Modify `AGENTS.md` (rewrite, under 60 lines).

Content: two sentences on what the project is with a link to the README; "Where the truth lives" table kept; "Picking it up after a pause" kept, shortened to four lines; "Where we are" replaced by one dated paragraph as of today (backend 0.20.0 deployed, firmware and app carry the three board latches on the wire, this readability pass in progress under the spec path); conventions kept, each one line, plus the comment policy from the spec. The repo state table and every diary entry go.

- [ ] Write the file.
- [ ] `wc -l AGENTS.md` under 60.
- [ ] Commit: `AGENTS: rules only, the diary goes`.

### Task 5: Org profile

**Files:** `profile/README.md` in a scratch clone of `git@github.com:plantbutler/.github.git`, branch `readable`.

Content: the umbrella README's first paragraph, the "start at the umbrella" line, the five-repo table with one line each. "rotary manifold" becomes the lead-screw manifold. Under 25 lines.

- [ ] Clone into the scratchpad, edit, commit `profile: match the umbrella README`, push, open a PR with `gh pr create`.

### Task 6: Newcomer test and PR

- [ ] Dispatch a fresh Sonnet agent with only `README.md`, `GLOSSARY.md`, the `DECISIONS.md` index and `AGENTS.md`. It must answer: what the system is, how the board reaches the backend, what a pot and a hose are, which repo to open to change the watering rule, how to clone. Every miss is a doc fix, then re-run.
- [ ] Fix the firmware description on GitHub (`gh repo edit plantbutler/firmware --description`), currently just "Plant Butler".
- [ ] Push `readable`, open the umbrella PR with the footer from the global instructions. Do not merge; report.
