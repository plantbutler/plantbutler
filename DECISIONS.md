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
