# Working on Plant Butler

Read this first, whichever repository you landed in. [README.md](README.md) says what the system
is and how its parts talk; [GLOSSARY.md](GLOSSARY.md) defines the words. Each repository has its
own short `AGENTS.md` with the tooling and traps specific to it.

## Where the truth lives

| question | answer |
| --- | --- |
| what is being built, in what order | `plan/`, one markdown record per pitch ([plan/AGENTS.md](plan/AGENTS.md)) |
| why it is built this way | [DECISIONS.md](DECISIONS.md), dated entries with an index, never edited in place |
| how to build, test, run a part | that repository's `README.md` |
| how to work in a repository | that repository's `AGENTS.md` |

## Picking it up after a pause

1. Read the index in [DECISIONS.md](DECISIONS.md) and any entry you do not remember.
2. `cd plan && openproj check . && openproj schedule .` to see what is bet and what is late.
3. Take the pitch that is `in_progress`, or the first `ready` one whose dependencies are done.
4. When you stop: update the pitch, commit in the repository you worked in, then commit the new
   submodule pointer here. Update **Where we are** below if the picture changed.

## Where we are

**2026-09-07.** Backend 0.20.0 runs on the NAS. Firmware and app carry the board's three latches
on the wire. A readability pass is under way: docs rewritten, comments cut to what a newcomer
needs, code and tests simplified, one repository at a time. Its spec is
`docs/superpowers/specs/2026-09-07-ease-of-understanding-design.md`; delete both when it is done.

## Conventions

- **Fat-marker plan.** A pitch is a few sentences per heading. Detail goes in the repository the
  pitch is about, once it is bet. No tasks, schemas or class designs in the plan.
- **Decisions are appended, not edited.** A changed decision is a new dated entry. An idea that
  might overturn one goes into `plan/notes/` first.
- **One record, one commit.** `openproj check .` reports no blockers before a plan commit is
  pushed. Never write a derived date into a record.
- **Secrets never enter a repository.** WiFi credentials, the token and the NAS address live in
  gitignored files or the environment. Nothing is ever port-forwarded on the NAS.
- **Failure direction is dry.** Any change to firmware or backend keeps decisions 5 and 7: when
  in doubt, no water.
- **Submodules.** After pushing in a subrepository, `git add <subrepo>` here and push the
  pointer, or the umbrella describes a state nobody can clone.
- **Other sessions edit these repositories at the same time.** One worktree per topic, one PR
  per repository, rebase before merging.

## Comments and docs

- A file starts with one line saying what it holds.
- A comment says why, not what. Keep invariants, units, hardware traps, security reasons.
- No dates, task numbers, PR numbers, pitch names, decision numbers, reviewer names or history.
  Keep the fact, drop the provenance.
- Docs are short and carry commands. Define a term on first use; the glossary is the reference.
