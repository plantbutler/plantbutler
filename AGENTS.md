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

**2026-09-04, later.** Both queued ideas are built, merged and deployed, and neither is tested on
a phone — adb is off at Jacopo's end. The two pitches stay `in_progress` for that reason: "Where
is the butler?" (backend#17, app#12) and "A picture of the plant, over time" (backend#19, app#13),
with the review fixes in backend#19 and app#12/#13 and the deploy in backend#20. Decisions 19 and
20 record what they settled. **The backend is 0.14.0 on the NAS**, verified over the tailnet
against a garden that is still empty.

Three reviewers went over the four PRs and found twelve defects between them, all fixed before
merging. The two worth remembering: a photograph whose id collided with an existing one overwrote
that picture's bytes before the INSERT could object, leaving a committed row pointing at nothing
(ids are claimed with O_EXCL now); and the setup screen could print the token it was refusing,
because OkHttp quotes an illegal header value back in its exception message and that message went
straight onto the screen. Three of the twelve were comments that lied about the code, and two of
those had a real bug sitting behind them.

The app no longer has the address or the token compiled into it: it asks on first start, proves
both with a real `GET /hello`, and keeps them in the phone's encrypted store, so one APK installs
on a second phone and a moved NAS is a typed line. `butler.properties` is now optional and only
prefills that screen for a development build. `GET /hello` had to exist because nothing else could
tell a wrong address from a wrong token — which are different mistakes and only one of them is the
user's to fix.

A pot also keeps its own photographs now: a strip under the chart, oldest first, with the care
source's picture of the species beside them as the reference. The bytes live under `BUTLER_PHOTOS`
next to the database and the row is the truth, which is what decides the direction a crash or a
half-restored backup fails in. The phone caps the long edge at 1600 before it uploads, and turns
the picture upright from EXIF first — a phone writes the sensor's orientation into a tag rather
than into the pixels, and re-encoding drops it, so without that every portrait photograph would
have come back on its side for good.

What is untested is exactly what only a device has: the camera, and a first start with nothing
stored. Both wait for adb.

**2026-09-04.** The second pass is finished. "What does this plant want?" was the last of its
pitches and it is done: backend 0.12.0 on the NAS, the app on the phone, and decision 18 recording
what it settled. `GET /species` resolves what somebody typed through GBIF and asks Trefle about the
accepted binomial; both hops are cached, so the garden screen never touches the network. GBIF knows
scientific names only, so "basil", "basilico", "tomatoe" and "peace lily" fall to Trefle's own
search, which matches common names, survives a typo, and answers with photographs — and the
photograph is what confirms the plant, because "peace lily" is two species and no spelling settles
which one is on the windowsill.

No watering number comes from any of it. Trefle carries no watering regime (`soil_humidity` NULL
for every species probed), so the target band is proposed locally from plant type, soil, pot size
and month, and it arrives as an offer with Apply and Not now. Applying it is an ordinary pot edit.

Driven on the phone against a laptop backend with a fake board: `basilico` returned eight
photographs, tapping Basil filled the field and re-resolved to *Ocimum basilicum* with light 7/10
and humidity 5/10, the saved species read its care back out of the cache with no second call,
Apply wrote 35-50% and the offer went quiet, and Not now silenced monstera's until a repot made it
a different offer (35-55%, "tropical, large pot").

Two ideas were queued and not bet at that point: photographs of your own plants filed under the
pot id, and asking where the backend is on first start instead of compiling it into the APK. Both
were built later the same day — see the entry above.

The hardware is still out of reach, so the two firmware pitches wait.


**2026-09-03.** Cycle 1's four backend pitches and the three app pitches are done and deployed;
the backend is 0.8.0 on the NAS, the app is a debug APK on the phone. A second pass on the app is
underway under two new projects, "App, second pass" and "The butler knows the plant". Two of its
six pitches are done. "A pot has an identity": a pot is now a `pot-xxxxxx` id rather than its name, its
wiring lives in `pot_mappings` with a validity window, and it carries a `species` — see decision
16. "The watering history": `GET /doses` and a screen for it, backend 0.9.0. That one also turned
up a defect older than itself — disabling a pot never closed its mapping window, so two pots could
hold one hose and a dose could belong to both, which had been quietly corrupting cooldown and
daily-cap attribution. Fixed in the mapping write. "Look further back": day, week and month chips
on the chart with a scrub, plus paging on the history. And "A create is not an edit", a pitch made
on the spot: an id-less `POST /pot` used to edit whatever pot already had that name, so a new pot
made against a stale list could silently overwrite one. The backend is 0.11.0 and deployed.

The app has been driven on the phone once, against a laptop backend with a fake board: a pot
created, renamed, watered, and read back in both histories. The rename is what the identity pitch
was for — three days ago the app could not do it at all.

"Something to look at off the tailnet" and "Three samples, not one" are done too, so five of the
six second-pass pitches are, plus the one made on the spot.

All five were then driven on the phone (2026-09-04) against a laptop backend and a fake board: the
chart's day/week/month chips and its scrub, the watering history paging on its cursor, the offline
cache cold-started with the backend killed, and the calibration wizard's three-sample capture with
its arm-and-restore. Two things only showed up on hardware and were fixed there.

The sixth, "What does this plant want?", is unblocked and reshaped. Probed with real keys on
2026-09-04: **Perenual's free tier is names only** — every care field is an upgrade string and the
detail endpoints answer 429 — and **Trefle answers with light, humidity and pH but has no watering
regime at all** (`soil_humidity` empty across every species sampled, and houseplants empty or
absent entirely). OpenFarm's API is gone; GBIF does the taxonomy hop free. Jacopo's call: Perenual
is out, no ranking and no fallback chain, Trefle is the one online source and otherwise the numbers
are typed in. The watering band was never going to come from a care API and now says so.

The hardware is still out of reach until about mid-September, so the two firmware pitches wait.

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
