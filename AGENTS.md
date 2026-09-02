# Working on Plant Butler

Read this first, whichever repository you landed in. It is the org-wide guide; each repository has
its own short `AGENTS.md` for the tooling and state that are specific to it.

Plant Butler is a hobby plant-watering system: an Arduino UNO R4 WiFi with one capacitive
soil-moisture sensor per pot and a pump feeding a lead-screw manifold (a servo moves a magnet cart that lifts one gate at a time); a small Python
backend on a Synology NAS that stores readings and decides when to water; an Android app to look
at the plants and water them by hand. Two workers: Jacopo (`jcanton`) and Claude (`claude`), a day
a week between them. Everything is deliberately small.

## Where the truth lives

| question                          | answer                                                                 |
| --------------------------------- | ---------------------------------------------------------------------- |
| what is being built, and in what order | `plan/` — the Shape Up plan, one markdown record per pitch ([plan/AGENTS.md](plan/AGENTS.md)) |
| why it is built this way          | [DECISIONS.md](DECISIONS.md) — dated entries, never edited in place    |
| how to work in a given repository | that repository's `AGENTS.md`                                          |
| what is bet right now             | `plan/cycles/0001.md` and the pitches with `status: ready`             |

The repositories, all under the GitHub org `plantbutler` and pinned here as submodules:

| repo       | what it is                                                        | state on 2026-08-30 |
| ---------- | ----------------------------------------------------------------- | ------------------- |
| `firmware` | PlatformIO / UNO R4 WiFi; the former `jcanton/plant_butler`       | prototype: reads A0-A3 onto two screens, cycles the manifold at boot |
| `backend`  | one Python container + SQLite on the NAS                          | README only         |
| `app`      | Kotlin + Jetpack Compose                                          | README only         |
| `cad`      | OpenSCAD, KiCad, BOM, bench notes                                 | README only         |
| `plan`     | the openproj plan                                                 | 16 pitches, cycle 1 bet |

Locally the umbrella is `~/projects/plant-butler/` with the submodules checked out inside it; the
firmware also has an older standalone checkout at `~/projects/plant_butler/` pointing at the same
remote. The tool that reads the plan is `openproj`, Jacopo's own, at `~/projects/openproj/`.

## Picking it up after a pause

1. Read [DECISIONS.md](DECISIONS.md). Ten minutes, and it stops you re-deciding things.
2. `cd plan && openproj check . && openproj schedule .` — what is bet, what waits on what, what
   is late. See `plan/AGENTS.md` for how to run `openproj`.
3. Find the pitch that is `in_progress`, or the first `ready` one whose dependencies are done.
   Its body says what done looks like. Open that repository and read its `AGENTS.md`.
4. When you stop: update the pitch (status, `## Progress` if it has one), commit in the
   repository you worked in, then update the submodule pointer here. Update the **Where we are**
   section below if the picture changed.

## Where we are

**2026-08-30.** The org, the five repositories (plus `.github`), the decisions, the plan and an
`AGENTS.md` in every repo exist; no product code has been written beyond the firmware prototype.
Cycle 1 (2026-08-31 → review 2026-10-12) is software only because the hardware is out of reach
until about mid-September. "Org, repos, decisions, shopping list" is `in_progress` with one item
left — order the bench parts; the platform is pinned and the old Gmail app password is revoked
and gone. Jacopo then
holds "Readings up the wire"; Claude holds "Readings land on the NAS" and "Command hand-off".
Starting either backend pitch needs access to the NAS (Container Manager or SSH); starting the
firmware pitch needs the board on USB.

## Conventions

- **Shape Up, fat-marker.** A pitch body is a few sentences per heading. Detail goes in the
  repository the pitch is about, once it is bet. Do not add tasks, schemas or class designs to
  the plan.
- **Decisions are appended, not edited.** A changed decision is a new dated entry in
  `DECISIONS.md`. An idea that might overturn one goes into `plan/notes/` first.
- **One record, one commit; check before push.** `openproj check .` must report no blockers
  before a plan commit is pushed. Never write a derived date into a record.
- **Secrets never enter a repository.** `firmware/include/secrets.h` is gitignored; the backend's
  token and the app's server URL live in untracked local files or environment. Nothing in this
  system is ever port-forwarded on the synology.me host.
- **Failure direction is dry.** Any change to firmware or backend keeps the invariants in
  DECISIONS.md #5 and #7.
- **Names.** Record ids are random; refer to records by title in prose. The org is `plantbutler`
  (`plant-butler` was taken); the local directory is still `plant-butler`.
- **Submodules.** After pushing in a subrepository, `git add <subrepo>` here and push the pointer,
  or the umbrella describes a state nobody can clone.
