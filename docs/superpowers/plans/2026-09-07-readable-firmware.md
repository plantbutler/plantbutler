# Readable firmware: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The firmware repository reads like a project a newcomer can build, flash and change, with comments that carry only what the code cannot say.

**Architecture:** Four PRs on branch `readable` in `.worktrees/readable-firmware`, in order: comments; code simplification with tests untouched; test simplification with code untouched; README and AGENTS.md against the final shape. Each PR merges before the next starts.

**Tech Stack:** PlatformIO 6.1.19, `pio test -e native`, `tools/check.sh`.

**Spec:** `docs/superpowers/specs/2026-09-07-ease-of-understanding-design.md` (umbrella).

## Global constraints

- `include/secrets.h` is gitignored and copied in by hand; never edit or commit it. `make test` needs it present.
- `tools/check.sh` greps comments as readily as code. A comment must never spell the name of something its file is forbidden to use (`Arduino.h`, the WiFi library, pin literals, `digitalWrite`); say it in words or say nothing. Run `make check` after every change. Its board-side checks need `pio run -e uno_r4_wifi -e uno_r4_wifi_bringup -e uno_r4_wifi_sim` and `pio test -e uno_r4_wifi_test -f test_device --without-uploading --without-testing` to have populated `.pio/build`.
- `make test` green (261 cases, 13 skipped at baseline) after every task.
- The dry-failure invariants hold: the single pump-pin writer in `safety.cpp`, the single watchdog feeder, the float debounce, the three latches.

---

### Task 1: Comment cleanup (PR 1)

**Files:** every `.h` and `.cpp` under `include/`, `src/`, `lib/`, `test/`. Also delete `include/README`, `lib/README`, `test/README` (stock PlatformIO templates) and `include/constants.h` (unused stub; confirm with grep first).

Policy, as given to each editing agent:
- File top: one comment line saying what the file holds. If the file has a real constraint (compiled only in one environment; must not include the board library because the hardware header is the only door), keep it as one clause.
- Keep: units, derivations, hardware facts, "the only writer/feeder" invariants, race explanations, why-not-the-obvious-way. Shorten each to the fewest words that still carry it.
- Delete: task numbers, fix rounds, spec section marks, commit hashes, plan or brief references, "used to", "was deleted here", what a future task will do, comments on include lines, comments that restate the code.
- Tests: a comment says what the test proves only when the name does not. Setup narration goes.
- Comments only. No code, whitespace or ordering changes on code lines.

Six batches, dispatched in parallel, each an editing agent that does not build:
- A: `include/*.h`
- B: `src/safety.cpp main.cpp exec.cpp noinit.cpp pulses.cpp sensors.cpp ui.cpp sim_console.cpp sim_console.h`
- C: `src/cli.cpp netfsm.cpp report.cpp hal_uno.cpp hal_sim.cpp link_fake.cpp`
- D: `lib/**`, `test/support/*`, `test/test_device test_cart test_sensors test_contra`
- E: `test/test_dose/test_dose.cpp`
- F: `test/test_net test_report test_cli`

- [ ] Dispatch A to F; each reports comment lines before and after per file and the comments it kept but was unsure of.
- [ ] `make test`; `pio run` for the three board environments; `make check`. Fix any comment that trips a grep by rewording it in words.
- [ ] Delete the three stock READMEs and `constants.h`; `make test` again.
- [ ] Sonnet reviewer over `git diff origin/main`: any deleted comment that carried an invariant, unit or hardware fact comes back, shortened.
- [ ] Commit per batch, push, PR "Firmware comments say why, not when".

### Task 2: Simplify code (PR 2, branch `readable-code` stacked on `readable`)

Decided from the survey, no test changes, each its own commit:
- `cli.cpp`: command table with help generated from it (one place to add a command); `cli_print_status()` split into named printers; the two digit parsers folded into one; the never-compiled `PB_RELAY_ACTIVE_LOW` arm deleted (also in `pins.h`).
- `tools/check.sh`: 706 lines to under 330, patterns and counts unchanged. `Makefile`: `test-all`, `test-device`, `build-all`. `platformio.ini`: wrong inheritance comment fixed, `-Wformat-signedness` on every device environment, duplicate `-I include` dropped, the dead by-time flag line dropped.
- `exec.cpp` and `config.h`: the backend by-time dose arm deleted (no environment compiled it; the console `pump` still doses by time). `PB_WDT_GRANTED_MS` as its arithmetic. `static_assert` that the rig dose ceiling is inside the protocol ceiling. `main.cpp`: one `boot_fail_()` helper, the two external accessors gone. `lib/Network/include/Network.h` deleted: `lib_deps` builds the library, the header was a comment in a file.

Deferred to the test step or later, because a test names them: `hal_boot_salt`, `pulses_test_tear_*`, `sensors_stuck` renames; the `link_fake_saw_*` tautology pair; the report body byte-budget as a `static_assert` chain; `net_poll()` and `dose_run()` extraction (`safety.o` must stay byte-identical between bench and bring-up); `PB_BUILD_NAME` to one home (check.sh counts `PB_BRINGUP` in exactly two files).

- [x] Read `~/.claude/skills/working-principles/principles.md` sections 1, 2, 5, 6.
- [x] Survey.
- [ ] Apply; `make test`, board builds, `make check` after each.
- [ ] Section 6 checklist over the diff. Reviewer pass. PR against `readable`; retarget to `main` once PR 1 merges.

### Task 3: Simplify tests (PR 3)

- [ ] Survey agent over `test/`: duplicated fixtures, cases that assert the same thing, setup that a helper in `test/support` already provides, dead skips.
- [ ] Apply; code untouched; case count may drop only where two cases proved one thing. Reviewer pass. PR.

### Task 4: README and AGENTS.md (PR 4)

- [ ] `README.md` under 80 lines: what it is, how it fits, `secrets.h` first, build, flash, monitor, test, sim, bring-up and calibration as commands, the environments table (which files compile where; `build_src_filter` replaces rather than extends), the file map one line per file, `make check` and its exit 2 on a clean clone, link to the glossary.
- [ ] `AGENTS.md` under 60 lines: the two seams and who may include what, the check.sh trap, the comment policy, operating cautions (12 V brick unplugged for sim; bring-up binary never left running).
- [ ] Delete `docs/superpowers/` and `CLAUDE.md` stays as `@AGENTS.md`.
- [ ] Newcomer test: fresh agent with README, AGENTS.md and the glossary answers build, flash, test, add a serial command, change a pin, change the dose limit.
- [ ] PR.
