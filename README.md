# Plant Butler

A plant-watering system: an Arduino UNO R4 WiFi with one capacitive soil-moisture sensor per pot
and a pump feeding a rotary manifold, a small backend on the Synology NAS that stores the readings
and decides when to water, and an Android app to look at the plants and water them by hand.

This is the umbrella repository. It holds no code: it pins the five repositories below as
submodules and keeps the documents that belong to the whole — this README, [DECISIONS.md](DECISIONS.md),
and eventually the NAS runbook.

| repo       | what it is                                                                | state                     |
| ---------- | ------------------------------------------------------------------------- | ------------------------- |
| `firmware` | PlatformIO / Arduino UNO R4 WiFi: read sensors, report, water on command  | today's `jcanton/plant_butler`, to be transferred as-is |
| `backend`  | one Python container + SQLite on the NAS: readings, pots, commands, rules | not started               |
| `app`      | native Android, Kotlin + Jetpack Compose                                  | not started               |
| `cad`      | OpenSCAD parts, KiCad wiring diagram, BOM, bench notes                    | not started               |
| `plan`     | the Shape Up plan, an [openproj](https://github.com/jcanton/openproj) repository | [plan/](plan/README.md)   |

## Working on it

The plan is the entry point: `plan/` says what is bet in the current cycle, what waits on what,
and why. Each pitch is one file — fields at the top, the shaping under it — and detail lives in the
repository the pitch is about, once it is bet.

```bash
git clone --recurse-submodules git@github.com:plantbutler/plantbutler.git
cd plant-butler/plan && uv run --project ~/projects/openproj openproj check .
```
