# Glossary

The words the three repositories use, one line each. Every document still defines a term the
first time it uses it; this is the place to check a meaning.

| term | meaning |
| --- | --- |
| ack | `ack=<id>` in a later report: the board confirms it received that command. |
| adb | Android's command-line tool for installing and driving an app on a phone over USB or the network. |
| advice | A target band the backend proposes from the plant kind, soil, pot size and month; the person applies it or not. |
| APK | The built Android app file that gets installed on the phone. |
| approve | A person saying yes to a proposal; it becomes a queued command. |
| auto mode | Per pot: the rules queue doses directly. The other mode is learning. Switching is a human act. |
| backend latch | The backend's mirror of a board latch: no rule waters and commands are refused until a person resumes. |
| band | A pot's target moisture range, a low and a high percentage. The rules water below the low end. |
| board | One Arduino running the pump, sensors and manifold. Also called controller; `c=` in every report. Board 0 is a real board. |
| calibration | Holding a pot's sensor in the air and then in water, so its dry and wet readings are known and a percentage means something. |
| canary channel | Channel 15, wired to nothing, read to catch a stuck multiplexer. |
| care source | Where a care number came from. No watering number ever comes from an outside service. |
| cart | The carriage a servo drives along the manifold; it lifts one gate at a time. Positioned by counted screw pulses and a home sensor. |
| channel | One sensor input on a board, reported as `chN=` raw counts. A few high numbers (204, 207, 210, 211) carry board status, not soil. |
| contradiction (contra) | The float says the tank has water but the meter counted nothing during a dose. The board stops itself. |
| contra latch | The board's own stop after a contradiction, cleared only by typing `clear contra` at its console. |
| controller | See board. |
| cooldown | The minimum hours between two doses for one pot. |
| daily cap | The most water one pot may get in a day. |
| dead-man URL | An outside URL the backend pings only after a fully clean alert pass, so a dead backend stops the pings. |
| dose | One watering: an amount in millilitres sent to a board as a command, then acknowledged and judged. |
| dose ceiling | The largest single dose the backend accepts and the board executes. One number, kept in both. |
| drop | The float going from water to empty; it closes one tank measurement. |
| dry latch | The `dry on` command at the board console. Survives resets, refuses every dose until `dry off`. |
| env pot | A pot whose name starts `env:`; it holds a room reading such as temperature, not a plant. |
| fail dry | The rule behind every safety choice: when in doubt, no water. |
| float | The tank's float switch, reported as `float=`. Watering needs three good samples in a row; one bad sample refuses. |
| flap | Three float refusals in a row. The board stops until a dose is granted. |
| garden | All the pots, as the app lists them. |
| gate | One of the manifold's outlets as the cart sees it: the position it parks over to water there. |
| GBIF | An outside service that turns a typed plant name into an accepted scientific name. |
| graveyard | Where a buried pot goes: records kept, wiring closed, no watering. Reversible. |
| HAL | Hardware abstraction layer: the header every hardware touch goes through, real on the board, faked on the host for tests. |
| hose | The water line from one outlet. Cooldowns and daily caps are counted per hose, not per pot. |
| k=v | The wire format both ways: space-separated `key=value` tokens. Report `c=0 ch0=8123 ch1=7902`; answer `next=60`. |
| latch | A stop that stays until a person clears it. The board has three (dry, contra, reset mid-dose); the backend mirrors them. |
| learning mode | Per pot: the rules propose doses and a person approves each. The other mode is auto. |
| manifold | The water distributor with one outlet per pot, so one pump serves five hoses. |
| mapping window | Which board, channel and outlet a pot was on, from when to when. Readings are attributed through it. |
| meter | The flow sensor. Its pulses are converted to millilitres and prove water actually moved. |
| NAS | Network-attached storage: the Synology box at home that runs the backend container. |
| noinit | The few bytes of board memory that survive a warm reset, holding the latches and a checksum. |
| ntfy | A public push-notification service. The backend posts alerts to a topic; the topic name is the secret. |
| openproj | Jacopo's command-line tool that reads the plan: checks it, schedules it. Lives outside these repositories. |
| outlet | One numbered water exit on the manifold, 1 to 5. Commands name an outlet, not a pot. |
| pitch | One shaped piece of work in the plan: problem, appetite, rough solution, a few sentences each. |
| PlatformIO | The build tool for the firmware; `platformio.ini` defines each build. |
| pos | Where the cart is: `pos=ok` or `pos=unknown`. Unknown blocks watering. |
| pot | One plant: its name, calibration and target band. A pot outlives its wiring. |
| proposal | A dose the rules want in learning mode, waiting for a person. Expires if ignored. Also called offer. |
| quiet hours | A daily hour range during which the rules never water. |
| report interval | Seconds until the board should report again, told to it in every answer as `next=`. |
| reset mid-dose | The board restarted while the pump was running. Latched so the fact survives the boot. |
| retired board | A board that is gone. Reports still land, nothing pages and nothing waters it. |
| seam | A header of plain functions with two implementations, board and host, chosen at build time. The firmware has two. |
| sim | The firmware build with no pump driver and no network, for driving the rig by hand. |
| slosh | One odd float or position reading, dismissed as water moving. Two in a row count. |
| soak | The wait after a dose before judging it, so the water can reach the sensor. |
| species lookup | Typing a plant name and getting back the accepted scientific name and what is known about it. |
| submodule | A pinned commit of another git repository, checked out inside this one. |
| tailnet | A private network between your own devices, made with Tailscale. The phone reaches the NAS through it. |
| tank | The water store one board pumps from. Its size is learned from full-to-empty runs. |
| target band | See band. |
| ticker | The backend's one periodic job, every minute: evaluate the alert rules from stored state. |
| Trefle | An outside service asked about a scientific name. It knows light and humidity, never watering. |
| uv | The Python tool that installs the backend's dependencies and runs its tests and server. |
| verdict | A person's recorded judgement of how a dose turned out. |
| X-Token | The one shared secret, sent as an HTTP header on every request by the board and the app. |

## Alert keys

The backend raises alerts under these key prefixes. Those marked once page once and never clear.

| prefix | raised when |
| --- | --- |
| `silent:` | A board that is not retired has not reported for longer than its threshold. |
| `sensor:` | A mapped channel stopped arriving while its board still reports. |
| `float:` | The float said empty twice in a row: tank empty, watering on hold. |
| `pos:` | The cart position was unknown twice in a row. |
| `fields:float:`, `fields:pos:` | A safety field the board used to send vanished and stayed gone. |
| `latch:` | The board stopped itself and waits for a person. Pages again when the reason changes. |
| `over:` | More than the tank holds was pumped and the float still says water: presumed stuck. |
| `stale:` | The float still says empty minutes after a recorded refill. |
| `tank:` (once) | A tank measurement differs from the recent median by more than the allowance. |
| `dose:` (once) | A dose was never acknowledged, came up short on the meter, or did not raise moisture after its soak. |
| `dosefail:` (once) | The per-board limit on dose failures: one page per board. |
| `proposal:` (once) | A proposal is waiting for approval. At most one nudge per hose per day. |
| `meta:` (once) | Bookkeeping: the observation window, and the occasional "backend is up" message. |
