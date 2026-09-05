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

## 2026-09-04

18. **One online care source, no ranking, and no watering number from any of it.** Trefle is the
    only service asked about a plant, with GBIF in front of it to turn what somebody typed into
    the accepted binomial. There is no second source and no fallback chain: if Trefle is offline
    or has never heard of the plant, the numbers are typed in, and that is a working path rather
    than a degraded one. The earlier ranking that put Perenual first was made from documentation
    and is overturned — with a real key its free tier answers `cycle`, `watering` and `sunlight`
    with the literal string "Upgrade Plans To Premium/Supreme… I'm sorry".

    The part worth keeping is what the probe found about Trefle itself. It carries no watering
    regime at all: `soil_humidity` was NULL for every species asked, houseplants and crops alike.
    So the target moisture band cannot come from a care source and never could. It is proposed
    locally, from what is actually on hand — the kind of plant, the soil, the pot size and the
    month — and it arrives as an offer. Accepting one is an ordinary `POST /pot`, which is what
    keeps "no watering number without a human approving it" literally true rather than merely
    intended. A refusal is remembered against the numbers refused, so a repot or a change of
    season is a new question and is asked again.

    Two consequences worth stating outright. Our percentage is a straight line between dry air and
    tap water, not volumetric water content, so no published figure could be copied into it even
    if a source had one — a care source gives regimes and context, and the numbers stay local.
    And for the plants this system is actually for, the answer is usually a picture and a name:
    Monstera deliciosa, Dracaena trifasciata and Chlorophytum comosum all resolve with every
    growth field empty. Which is why the lookup ended up being a search with photographs rather
    than a table of numbers — GBIF knows scientific names only, so a common name, another
    language or a typo falls to Trefle's own search, and what confirms the plant is that you
    recognise it.

    Numbered 18 and not 17: another session has 17 in an open PR, and a gap is cheaper than a
    collision.

19. **The address and the token belong to the device, not to the build.** The app asks where the
    butler is on first start, proves the answer with a real call, and keeps it in the phone's
    encrypted store; a settings screen changes it later. `butler.properties` survives only as the
    default a development build prefills that screen with, and the build no longer requires it, so
    an APK built without it carries no token at all. That is the point rather than a convenience:
    every artifact ever produced had the token inside it, a second phone needed a rebuild, and a
    moved NAS needed a new build.

    Proving it needed a new route. Nothing that existed could tell a wrong address from a wrong
    token: the reads are ungated and answer a wrong token exactly as they answer a right one, and
    every gated route writes something. So backend 0.13.0 has `GET /hello` — token-gated, touching
    no database, answering `butler=<version>`. Probing authentication with a deliberately invalid
    write would have worked today and broken silently the day a route checked its body before its
    token, and touching no database means a butler whose volume came unmounted can still say the
    token was wrong.

    Those are different mistakes, only one of them is the user's to fix, and saying which is most
    of what the screen is for. Two rules fall out of the same place: a scheme the user did not type
    becomes http and never https, because the NAS is plain HTTP on the tailnet and cleartext has
    to keep working; and the offline cache belongs to one butler, so it is cleared on a repoint
    *and* stamped with the address it came from, since a delete that failed would otherwise show
    one server's garden under another's name. The rabbit hole about the client no longer being
    built once at start-up has a second half nobody wrote down: an answer already in the air from
    the old address. Every request now runs under one cancellable job, and repointing drops it.

20. **A photograph's row is the truth; its bytes are only bytes.** Pictures of a plant hang off the
    pot id, one file per row under `BUTLER_PHOTOS` beside the database so the two are backed up or
    lost together. A picture is listed, served and deleted by its row, and the directory is never
    read to decide what exists.

    That is what settles the only genuinely hard question here, which is what a crash or a
    half-restored backup is allowed to leave behind. A file no row knows about is invisible and
    harmless. A row whose file has gone cannot be hidden, so it is reported as missing rather than
    served as an image that will not load. Keeping a picture therefore writes the file and then
    the row; deleting one removes the row and then the file. Both orders leave the harmless
    direction, and deleting favours the person's intent over the volume's cooperation.

    Two smaller things worth fixing in writing. These four routes are the only gated *reads* the
    backend has — everything else here is numbers about plants, and a photograph is the one thing
    that could show the inside of a house. And the phone caps the long edge at 1600 and re-encodes
    before it uploads, which is the difference between a decade of weekly pictures being a couple
    of hundred megabytes and a couple of dozen gigabytes on a NAS volume that was never sized for
    either.

    A pot outlives its plant. Rather than invent a replant event, each row carries the species the
    pot had that day, and the strip draws the break where it changed — which is honest about what
    it cannot see: basil replanted with basil leaves no trace, and neither does a plant that was
    never named.

21. **A pot's size is a measurement, and the shift it earns is the log of the volume.** The two
    size fields were free text and neither did what its name promised. `pot_size` was matched on
    the words *small* and *large* alone, so `14cm` — the README's own example, and the `"3"` a
    real pot was created with — moved the band by nothing at all; `plant_size` was read by
    nothing whatsoever. They are now `pot_diameter_cm` and `plant_height_cm`, REAL, and the pot is
    read as what it physically is: a water store.

    Volume goes as the cube of the diameter, and the temptation is to make the shift cubic too.
    That is wrong by a wide margin — a 40 cm pot holds 23× the water of a 14 cm one, and no
    percentage band survives being multiplied by 23. What is linear in percentage points is the
    *log* of the volume: each doubling of buffer moves the band one step, 2.5 points, so the cube
    survives as the factor of three in `log2((d/14)**3)`. That lands 10 cm at +4 on the floor and
    24 cm at −6 on both ends, which is where the old *small* and *large* keywords sat, and gives
    every size between them an answer for the first time. It saturates at three doublings — a
    28 cm pot and a 60 cm one get the same shift — which is not a claim that they want the same
    water but an admission of where a table of six plant kinds stops being worth extrapolating.

    Height is read *over* diameter and never on its own, because 40 cm of basil is thirsty in a
    10 cm pot and comfortable in a 30 cm one: it is demand against that buffer. Both effects move
    the floor; only a big pot moves the ceiling, and downwards. Nothing lifts a ceiling, because
    no container is a reason to keep a plant wetter than its own kind wants — the same direction
    #5 asks for.

22. **The kind of plant is a closed set, and a lookup may pre-select it.** `plant_type` was free
    text keyword-matched against eight words while being the only thing that picks the base band:
    an unlabelled plant starts at 35–55%, a succulent at 15–30%. So `plant_type=basil` was saved,
    matched nothing, and cost twenty points with nothing on screen to say so — the one failure
    mode a form should never have. It is now one of six kinds, refused on write and still tolerated
    on read, because rows written before the set existed must stay readable and the base band is
    the honest reading of one. *Not sure* stays a real answer with a real band; it is the correct
    state for a cutting somebody handed you.

    That closed set is also the door through which a species finally reaches a watering number.
    Decision 18 stands — no care source carries a regime and none ever sets a band — but GBIF's
    reply already contains the botanical family, free and keyless, and family maps onto these six
    kinds well enough to open the dropdown pre-selected. It is a guess and is treated as one: it
    fills the field only while the field is empty, and one tap changes it. Wrong costs a tap;
    silent costs twenty points.

    Genus is asked before family, because family is wrong exactly where it matters — Asparagaceae
    holds *Dracaena fragrans*, a leafy thing that wants watering, and *Dracaena trifasciata*, a
    succulent in all but name. Orchids are left unguessed on purpose: an epiphyte on bark waters
    nothing like a flowering pot plant, and falling through to *not sure* beats twenty confident
    points in the wrong direction.

23. **`schema.sql` stays additive, but `CREATE TABLE IF NOT EXISTS` is not.** It is additive about
    *tables*: a column appended to a CREATE that has already run never reaches an existing
    database, and the table quietly keeps the shape it was born with while every fresh one gets
    the new shape. That is how two of these fields would have gone out working perfectly in the
    tests and not at all on the NAS.

    So a new column goes in the CREATE *and* in `butler.ADDED_COLUMNS`, which ALTERs in whatever a
    given database has not got yet, and may carry a value over from an old column — `"14cm"` and
    `"10"` came across as measurements, `"small"` was dropped rather than invented into
    centimetres. Deliberately an ALTER and not a second rebuild: the one rebuild this project has
    (`migrate`, decision 16) stays the only one. Nothing is ever dropped, so a rollback to the
    previous container still reads every pot it wrote.

    `pots_now` is the exception that proves it: dropped and recreated on every start, because a
    view holds no data — it is derived, like a percentage — and `IF NOT EXISTS` would leave an
    older database serving the old shape forever with nothing to say it had. The ordering matters
    and is the whole trap: SQLite creates a view without checking that its columns exist, so the
    ALTERs must run *before* the script, or a perfectly successful startup is followed by every
    single read failing.

## 2026-09-05

24. **A pot has a status, not a switch, and the graveyard is what unwires it.** `pots.enabled`
    becomes `pots.status`, a closed set of `alive` and `graveyard` shaped so a third word — paused
    but still wired, say — is one entry plus one label. Every reader asks a positive allow-list
    (`butler.waters`, `butler.live_sql`) and never `!= 'graveyard'`, so a status a newer backend
    invents waters nothing, proposes nothing and pages about nothing in an older reader. Failure
    direction is dry, and that applies to words as well as to readings.

    Burying a pot CLOSES its open `pot_mappings` window, which is what actually frees the channel
    and the outlet; it also expires the pot's open proposals and drops its `sensor:` and
    `proposal:` alerts. That last one is not tidiness: both the raise and the clear for
    `sensor:<c>:<ch>` live inside a loop over the live pots, so a pot that leaves the loop while
    its alarm stands leaves a row nothing can ever clear, sitting in `/health` and inflating the
    daily up-probe count for good.

    This overturns the note under decision 16 that a disabled pot "does not unplug it". That was
    true and was the bug: the window stayed open, so a disabled pot still held a hose it was not
    using, and the displacement backstop existed only to clean up after it. Restoring a pot leaves
    it unwired, because the plant that comes back is not in the socket the old one left.

25. **A pot can be erased, and the command log is no longer never-pruned.** `POST /pot/delete`
    removes a pot, its wiring, its readings, its doses and their verdicts, its dismissed advice
    and its photographs with their files. Its own route rather than a field on `POST /pot`, for
    the reason `/photo/delete` is: a save that lost its body must never become an erasure.

    It overturns `schema.sql`'s "rows are never deleted" for `commands`. That rule was written
    against pruning, not against an owner asking for a plant to be gone, and the cost is real and
    named here rather than discovered: a deleted pot's doses stop floating the hose-keyed cooldown
    and daily-cap floors, so the next pot on that hose can be watered sooner than decision 5 would
    like, for up to a day.

    The order inside the transaction is forced by reachability — there are no foreign keys in this
    database and nothing cascades — and two of its steps are correctness rather than housekeeping.
    `commands.id` is a rowid alias with no `AUTOINCREMENT`, so SQLite hands the same ids out
    again: a leftover `verdicts` row would label a stranger's dose, and a leftover `dose:<id>`
    alert would make the judgement loop skip a real dose for ever on its `NOT EXISTS` guard. Both
    must be deleted through the commands they are found by, so both go first. The photograph files
    are unlinked only after the commit, and the pot's directory with `rmdir` rather than `rmtree`,
    because the directory is not the truth and a tree delete would take bytes belonging to rows
    the transaction never saw.

26. **Attribution is stamped, not derived.** `readings.pot_id` and `commands.pot_id` are written
    as the row lands, from the `pot_mappings` window in force at that moment, and `GET /history`
    takes a pot rather than a channel. This is what stops a plant wired into a dead one's socket
    opening its chart on somebody else's moisture curve — the case the whole pitch exists for.

    It is a reversal of half of decision 6, and only half. Percentages stay derived: recalibrating
    still re-reads the whole history. Attribution does not, and the price is that a mis-typed
    channel corrected later leaves its readings stamped with the mistake, where the window join
    would have re-attributed them. That was judged worth it — a wrong socket is caught in minutes,
    a chart quietly showing a dead plant's soil is not caught at all.

    `pot_mappings` is demoted from "the join that answers whose dose it was" to "the source the
    stamp is read from, and the record of which sensor a pot was on". The dose judgement still
    reads it for the channel, because the pot may have been rewired between the dose and the soak
    and the rise belongs to the probe that was in that soil at the time — a different question
    from whose dose it was, and it was only ever answered by accident that the same join did both.

    What stays keyed on the hose stays, and is now load-bearing for a second reason: the board is
    handed an outlet and not a pot, a proposal is fenced to whoever is on that hose now, and both
    watering floors count what went down a hose whoever it was attributed to. A command whose
    stamp is NULL — a hose no pot was on — is invisible to every pot-keyed gate and visible to
    those floors, which is the only thing keeping decision 5 true for it.

    One divergence, recorded rather than fixed: a proposal made for a pot, then the pot is
    rewired, then the stale proposal is approved and handed down the old hose, is stamped with the
    pot it was made for. The read-time join would have called it the new occupant's. It is safe in
    the dry direction because the hose floors still see it.

27. **Twelve plant kinds, seven soils, and free text is gone from the band engine.** The kinds go
    from six to twelve, all with distinct bands: a cactus leaves `succulent` because 5 points is
    the difference between a barrel cactus and an echeveria, and Orchidaceae stops being the
    deliberate omission of decision 22 now that there is a band for a bark epiphyte to have.

    Soil becomes a closed set too, refused on write like the kind, and only the seven values that
    actually move the band are in it. An ordinary potting mix is deliberately absent: it is what
    the plant kinds are written against, so "not said" and "the bag from the shop" are the same
    answer, and a list of movers stays a list of movers. Every ceiling shift is `<= 0` and that is
    an invariant, not an accident — the band only ever widens downwards, so a soil that raised the
    ceiling above the plant's own base would contradict the reason printed beside it.

    `_find` goes with them, and its three rules with it: the word-start prefix that kept a
    cauliflower from being a flower, the negation rule that made "not sandy" not sandy, and the
    phrase-as-reason that printed the user's own words when they were short enough. Each of them
    was a wrong answer free text had produced. A closed set cannot be misspelled, so the rules
    that survived misspellings have nothing left to do — recorded here because the code that
    explained why they existed goes with them.

    Reading stays tolerant in both fields, as it was for the kind: a row still holding free text
    reads fine and simply shifts nothing.

28. **The controller is an integer, and board 0 is a real board.** `c=` was the last free-text
    identifier on the wire, and the one a typo could turn into a whole second garden: a report
    from `bench1 ` or `Bench1` opens its own row in `controllers`, its own heartbeat and its own
    alerts, and nothing anywhere says the two are the same board. It is `0..255` now
    (`MAX_CONTROLLER`), in every parser and every column, and the firmware's `PB_CONTROLLER` is an
    integer asserted into that range at compile time.

    The half worth writing down is board **0**. It is falsy in Python and in Kotlin, and it is the
    number a new pot's form fills in, so `if not controller` refuses the commonest board there is
    — with "no c= in the report", which is the least helpful message it could have picked. Every
    check is `is None`, and every one of them has a test that fails without it. The same rule
    reaches the app: nothing tests a controller for truth, and `boardName()` is the single place
    that turns the number into "board 0" for a person, because a bare "0 has gone silent" reads
    like a truncated sentence.

    The database had to be recreated rather than migrated, and not for tidiness: `CREATE TABLE IF
    NOT EXISTS` cannot retype a column (decision 23), so the old TEXT affinity would have stored
    the integer as `'0'` and answered `/pots` a JSON *string* — which the app, with `isLenient`
    off, refuses to decode into an `Int`, failing the whole garden fetch rather than one field.
    Comparisons would have kept working, which is what makes it the bad kind of bug.

    What did NOT change: the alert keys still embed the controller as text (`silent:0`), because a
    key is a string and always was, and the app reads those keys rather than parsing them.

29. **The dose ceiling is one number in two places, and the backend latches on the board's word,
    not on an empty float.** `MAX_DOSE_ML` in butler.py is 250, the firmware's
    `PB_DOSE_RIG_MAX_ML`, refused at `POST /command` and at pot save; the board keeps its own copy
    as the backstop (#5), and the two move together. A backend that accepted 1000 against a board
    that refuses above 250 queued a dose that was refused, acked with nothing, cooled down and paged
    high, once per cooldown, forever — and never watered. Per-controller ceilings wait for a second
    kind of rig.

    The firmware's contradiction latch lives in `.noinit` and a power cycle erases it, so the
    backend holds the durable half: it latches on `ch207=1` or `err=contra`, and on `err=resetmid`,
    and stays latched — the rules go dry and `POST /command` refuses water — until `POST /resume`
    from the app, beside the words "type `clear contra` on the board". The float going empty does
    not latch: the rules already refuse on `float=0`, a float that goes 0 → 1 across a refill is
    demonstrably moving, and the refill button is the human event for an empty tank. A refill that
    the float does not move across is what pages instead (`ch204` against the refill log). And the
    daily cap charges acked water only: a lost response is likelier than a lost ack — the board never
    retries once response bytes arrived — and the cooldown still spaces the doses.
