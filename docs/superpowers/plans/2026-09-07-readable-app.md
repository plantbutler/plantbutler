# Readable app: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A newcomer can build and install the app, and find where to add a screen, a backend call, a form field or a setting.

**Architecture:** Four PRs on branch `readable` in `.worktrees/readable-app`, in order: comments; code simplification; test simplification; README and AGENTS.md.

**Tech Stack:** Kotlin, Jetpack Compose, Gradle, JUnit on the JVM.

**Spec:** `docs/superpowers/specs/2026-09-07-ease-of-understanding-design.md` (umbrella).

## Global constraints

- `JAVA_HOME=/opt/homebrew/opt/openjdk@17 ./gradlew --offline test` green after every task: 694 tests, 0 failures at baseline.
- A fresh clone needs `local.properties` with `sdk.dir`, or `ANDROID_HOME`. It is gitignored and the failure message is only in the Gradle output. The README must say so first.
- No behaviour change: what each screen shows, every request the app makes, every stored key.
- The address and the token stay out of the build and out of git.

---

### Task 1: Comment cleanup (PR 1)

Policy per the spec. Twelve files open with a rationale paragraph; each becomes one line saying what the file holds, with the rationale kept only where it names a trap.

- [ ] Agent A: `GardenViewModel.kt`, `Backend.kt`, `Settings.kt`, `Cache.kt`, `Main.kt`.
- [ ] Agent B: the screens: `GardenScreen.kt`, `PotScreen.kt`, `PotForm.kt`, `CalibrateScreen.kt`, `SetupScreen.kt`, `DosesScreen.kt`, `CareScreen.kt`, `PhotoStrip.kt`.
- [ ] Agent C: the pure files: `Garden.kt`, `Chart.kt`, `Water.kt`, `Doses.kt`, `Care.kt`, `Photos.kt`, `PhotoFile.kt`, `Calibration.kt`.
- [ ] Agent D: `app/src/test/` (14 files, about 5,300 lines).
- [ ] Keep: Android and Kotlin traps (reference equality on a byte array; the orientation tag a re-encode drops), the security reason a message is not printed, units, clock invariants. Delete dates, version numbers, pitch quotes, "used to".
- [ ] Tests, then a reviewer over the diff. PR.

### Task 2: Simplify code (PR 2)

- [ ] Survey first.
- [ ] `GardenViewModel.kt` is 1,205 lines with sixteen concerns and no section markers: split by concern into files that keep the same public surface, or mark sections if a split would spread one state flow across files. Decide with the survey in hand.
- [ ] The pot form's field logic is spread across three places with an implicit contract: the field table, a `when` over field keys in the screen, and the wire field list. Make adding a field one place, or name the contract where a reader meets it first.
- [ ] Renames where the name lies: `flight`, `strip`, `rowNote`, `Issued`, `doseWho`, `readHello`.
- [ ] State the file convention in one line where a newcomer meets it: screens in `*Screen.kt`, pure logic in the bare-noun file so the tests on the JVM can reach it, the wire in `Backend.kt`.
- [ ] Reviewer. PR.

### Task 3: Simplify tests (PR 3)

- [ ] Survey `app/src/test/`. Known: `GardenViewModelTest.kt` is 1,409 lines and 73 cases, and holds the screen behaviour tests that have no file named for them.
- [ ] Apply; code untouched. Reviewer. PR.

### Task 4: README and AGENTS.md (PR 4)

- [ ] `README.md` under 80 lines: what it is, how it fits, the SDK requirement first, build, test, install, point it at a backend on first start, the screen map, the file map, a link to the glossary.
- [ ] `AGENTS.md` under 60 lines: the file convention, the comment policy, the traps, how to drive it on a phone.
- [ ] Delete `docs/superpowers/` after the README absorbs the working test command and the wire contract it alone records.
- [ ] Newcomer test, then PR.
