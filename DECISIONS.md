# Decisions

Architecture decisions that the plan hinges on. One entry each; a decision that changes gets a new
dated entry, not an edit. Ideas that might overturn one go into `plan/notes/` first.

## 2026-08-30

1. **Repositories.** A GitHub org `plantbutler` (`plant-butler` was taken) with one repository each for `firmware` (the
   existing `jcanton/plant_butler`, transferred as-is — its history never held a secret), `backend`,
   `app`, `cad` and `plan`, pinned as submodules by this umbrella.

2. **Backend on the NAS, LAN-only.** One Python container with SQLite on a bind-mounted volume in
   Docker on the Synology. Nothing in this stack is ever port-forwarded on the synology.me host: one
   static token and an endpoint that turns on a pump are not internet-facing. Alerts leave via a
   public ntfy.sh topic so they reach the phone off-LAN; Tailscale is the remote-access option for
   later.

3. **Android app, native.** Kotlin + Jetpack Compose. The phone talks only to the backend.

4. **Board ↔ backend protocol.** Board-initiated plain HTTP, one round trip per report interval,
   `k=v` lines each way — no JSON, no MQTT broker, no server on the board. The response carries the
   next interval and at most one pending command. Chosen over MQTT because a broker is one more
   container, retained commands replay, and keepalives drop during multi-minute blocking manifold
   moves.

5. **The backend decides, the firmware protects.** Watering logic (thresholds, cooldowns, daily
   cap, quiet hours) lives in the backend so it can be edited without a reflash and has one source
   of truth. Firmware keeps only the invariants that must hold when the backend is wrong: max
   seconds per dose, minimum gap since boot, pump off on boot, on error, when the manifold position
   is unknown, and when the float switch says empty. NAS or WiFi down means no watering: **fail
   dry**, in writing.

6. **Raw on the wire, identity and calibration in the backend.** The board reports
   `(controller, channel)` raw counts and accepts a valve index; the backend maps channel → valve →
   pot → plant, so repotting is an edit. Two calibration numbers per channel, editable from the app;
   percentages are derived at read time and never stored, so recalibration reinterprets history
   instead of losing it. The two I2C screens are dropped (they were debugging aids; A4 becomes
   channel 5), ADC resolution is picked once (14-bit).

7. **Safety is layered and fails dry.** Actuators on their own supply with a common ground; a
   driver that is off unless the MCU actively asserts it; a float switch both in the driver circuit
   and on a sense pin; a reservoir small enough that a full dump is a mop-up; a watchdog. Unattended
   automation is gated on the interlock pitch.

## 2026-09-02

8. **Hardware is OpenSCAD, parametric, printed flat.** Every part lives as OpenSCAD source in
   `cad/`, every dimension derived from one parameter file with asserts; the FreeCAD assembly
   (valveV2) is kept as read-only reference, not edited. Chosen because the FreeCAD spreadsheet
   parametrics did not derive cleanly, because text renders headless and diffs, and because the
   owner and the agent both need to see sections, not trust them. Every part prints on the
   Bambu P2S with one flat face on the bed and no supports.

9. **The manifold mechanism.** A continuous-rotation SG90 turns an M3 lead screw through two
   spur gears outboard of the servo plate; a cart with a ø6×3 magnet rides on the lid and lifts
   one gate at a time; a gate is an 8 mm 440C ball on an O-ring seat in a cage, lifted enough
   that the throat under it passes `lift_factor` × the outlet bore area; the cart nut is
   captured; the lid and caps are bonded. Positioning is hall pulse counting on the screw plus
   the threadless home (decided in July, kept). Magnetized spheres, nail poppets and a rotary
   distributor were rejected.

## 2026-09-03

10. **The bench electronics use the parts in hand, and the interlock moves into firmware.**
    A one-channel relay module switches the pump's 12 V + leg: a de-energised coil is an open
    contact, so reset, boot, a pulled jumper and lost power all leave the pump off in hardware.
    No MOSFET, no 74HC00, no flyback diode (the module carries its own coil diode and the
    contacts take the motor's kick as an arc). No UBEC either: the UNO R4's ISL854102 buck gives
    1.2 A total from VIN, enough for the servo and the coil on the board's own 5 V pin, provided
    the board is fed from the barrel jack rather than a USB port and C1 (470-1000 µF) sits at
    the servo's plug. One 5 V rail.

    The cost is explicit: nothing in hardware now ANDs "firmware says pump" with "the tank has
    water", so a sketch that hangs with the pump pin asserted keeps pumping. Three firmware
    measures stand in for the gate and all three are mandatory — the RA4M1 IWDT enabled, a hard
    maximum run time in the same code path that asserts the pin, and a no-flow abort from the
    meter. The hardware gate returns with "Don't flood the flat" when the parts arrive.

11. **Signals go where their type belongs, not where the expansion story is tidiest.** Analog
    and slow (moisture, light) on the analog mux, 16 a piece. A level and slow (the manifold's
    home hall) on the I2C expander. A counted pulse train (the screw hall, the flow meter) on an
    interrupt pin, because a mux sitting on another channel when the edge arrives loses the
    count, and a lost screw edge is lost cart position. Timed one-wire (DHT11) and a servo's
    50 Hz train on direct pins. The float, the input that refuses a dose, takes the shortest
    path there is: a direct pin. Past three manifolds the screw halls route through a 74HC4051
    into one interrupt pin (only one manifold moves at a time) and the servos onto a PCA9685.

12. **The float is mounted so that "allowed" is the active state.** A stop caps the float's
    upward travel at the trip level and the hall sits at that stop, so for any level above the
    line the float is held against it and its magnet is at the sensor. Magnet present therefore
    means water above the line, and a dead, unplugged, unpowered or cut-off hall reads as
    refuse. The easier arrangement — a free float and a hall low in the tank — was rejected
    because it can only see "empty", which makes permission the passive state that every broken
    sensor also reports.

13. **The bench firmware is one sketch with two seams and one door to the pump.** `hal.h` and
    `link.h` are the only two boundaries: exactly one translation unit includes `<Arduino.h>`,
    names a pin, owns an ISR or writes the pump line, and exactly one names WiFiS3. Everything
    else compiles and is tested on the host. `dose_run()` is the only caller that writes the pump
    pin, with the cap three lines above the assert and no `return` between the ON and the OFF
    write; `safety_tick()` re-asserts the pump's rest state and only then feeds the watchdog, so
    the dog cannot be fed without the pin having just been put back where it belongs. The
    fake-hardware mode is a different file chosen at link time rather than a runtime flag, so the
    file that names the pump pin is not compiled into it. A handful of greps in `make check` turn
    those invariants into build failures instead of conventions.

    Three things this settles that were previously assumed. The wiring README's recipe
    `digitalWrite(D6, OFF); pinMode(D6, OUTPUT)` is **wrong for this silicon**: the core's
    `pinMode` is a whole-register write that discards the preceding level, so on an active-LOW
    relay module it would assert the pump at every boot. Direction and level go in one
    `R_IOPORT_PinCfg` call. The bring-up console ships in its **own binary**; `pump`, `prime`,
    `hang` and `cal` are compiled out of the one left running unattended. And `pos=unknown` is
    forced by a constant that ships defined, so the backend's watering rules stay dark until
    flipping it is a deliberate act after the 48-hour run.

    What the board runs is the core's **WDT**, not the IWDT that decision #10 and the wiring
    README both promise: the Arduino core exposes no other, the granted window is 5592 ms, and
    `status` says which dog is running. The real IWDT needs a flash option byte that can lock the
    board out of USB uploads, so it is its own piece of work with a proven DFU recovery path as
    its first deliverable.

14. **The float is checked by three independent witnesses, and a refusal refuses watering only.**
    The mounting is the first (decision #12): permission is the active state, so every electrical
    failure reads as refuse. It cannot catch a magnet that has come off the float and stuck to the
    hall's housing, which reads "full" forever on an empty tank. So there are two more. The flow
    meter is the witness for *right now*: a dose that the float permitted and that produced no
    flow at all means two sensors contradict each other, and the board latches watering off until
    a human clears it at the console — a kinked hose latches too, and that is intended, because
    the latch refuses rather than diagnoses. The human refill log is the witness over *time*: a
    refill is an act the board cannot observe, so only the backend can tell a float that has not
    moved in a month from one that is stuck, and only the backend's own latch survives a board
    reset. The board's whole contribution there is one reported number, seconds since the float
    last changed state, and it never refuses on staleness.

    A refusal stops a dose and nothing else. No latch stops a sensor read, a report, a retry, an
    ack or a screen. A latch is reached exactly when the backend most needs data — it is how the
    phone learns there is a problem — and a board that stopped reporting would look identical to a
    board that had died.

15. **The bench keeps both screens, so #6's "A4 becomes channel 5" is superseded.** Decision #6
    dropped the OLED and the LCD as debugging aids and freed A4 for a fifth channel. The bench rig
    keeps both: standing next to a pump you want to read what the board thinks without a laptop
    attached, and the channel they were costing is no longer theirs to cost — A4/A5 are the I2C bus
    that carries the mux select lines and the home hall, and the five channels arrive through the
    mux on A0. Both screens sit on that same bus at 0x3C and 0x27, alongside the expander at 0x20.

    Keeping them is not free and the price is paid in the one place it matters. Every screen paint
    is bus traffic on the wire the mux select lines and the home hall depend on, and one 16-character
    LCD row costs 96 Wire transactions; so neither screen is painted while the pump pin is asserted,
    and the refresh cadence is deliberately coarse. Any library added to this bus has to be checked
    for `Wire.flush()`, which spins without a bound and would hang a dose. The rest of #6 stands:
    raw counts on the wire, identity and calibration in the backend, 14-bit picked once.

16. **A pot is an id, not a name, and its wiring is something that happened over time.** Every pot
    gets a random `pot-3f9a21` that never changes, so the name becomes a nickname the app can edit
    and renaming is a field edit rather than disable-and-recreate. The controller, channel and
    outlet leave the pots table for `pot_mappings`: one row per period a pot spent on a given
    wiring, the open row (`to_ts IS NULL`) being where it hangs now. A view `pots_now` joins the
    two and hands every existing reader the column shape it already expected, so moving the wiring
    out cost one word at each of six call sites rather than a join apiece. `species` arrives in the
    same rebuild because the table was open anyway, and it is what the care lookup will key on.

    Two prices, both paid deliberately. SQLite cannot retype a primary key in place, so this is a
    one-time rebuild and therefore the single declared exception to `schema.sql`'s additive-only
    header — a function, not a framework: atomic, idempotent, announcing on stderr what it did, and
    leaving `<db>.pre-identity.bak` behind. That backup is taken after a `wal_checkpoint(TRUNCATE)`,
    because the live database runs in WAL mode and a plain file copy would preserve everything
    except the commits most at risk; if another connection holds the log open, the rebuild refuses
    rather than copy a short backup. And a dose now belongs to the pot that held that hose when the
    board was handed it, not to whoever hangs there now — which repaired a verdict log that was
    quietly filing one pot's judgement against another pot's dose, in the table that is meant to be
    the dataset adaptive dosing will one day fit on.

    What the second of those cost, and the correction, because it is the more useful half of this
    entry. Keying the cooldown and daily-cap gates to the pot made both depend on attribution
    succeeding, and a dose the lookup could not place read as "never watered" rather than "wait".
    That is fail-open, against #5, and it needed no exotic state to reach: prime a line by hand,
    then register the pot that hangs on it, and the rules queue a second dose into a pot that was
    watered a minute ago. The gates are now a union — the pot-keyed lookup with the old hose-keyed
    one underneath it as a floor — so a dose nobody can attribute still refuses. An unattributable
    dose is evidence that water came out, and evidence that water came out is a reason to wait.
