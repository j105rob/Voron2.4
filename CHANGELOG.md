# Voron 2.4 350 — Change Log

Changes to the printer (`voron.local` / `192.168.4.51`) and this repo, newest first.

**Workflow for config changes:** edit in the repo → check the live file still matches the repo → commit and push to GitHub → back up the live file → upload through Moonraker → restart Klipper (never mid-print) → check for warnings, and roll back if anything fails.

---

## 2026-10-13

### Flow test — OrcaSlicer "Max flowrate" (no config change)
- ABS at 255°C, Revo 60 W, standard 0.4 nozzle. Test settings: start 8, end 20, step 0.5.
- Quality starts to drop at **23 mm**, so the limit is 8 + 23 × 0.5 = **19.5 mm³/s**.
  - The drop was gradual, and the Clockwork 2 made no clicking or skipping.
- Recommended filament **Max volumetric speed: 16 mm³/s** (about 82% of the limit; was 12).
  - This leaves margin for long hot prints, where Clockwork 2 users report slipping from about 18 mm³/s.
- At 0.25 mm layers, 16 mm³/s allows about 160 mm/s on 0.40 lines and about 140 mm/s on 0.45 infill lines.

### Input shaper retuned — commit `3288d74`
- File: `config/voron-24-350.cfg` (`[input_shaper]`)
- **Why:** after re-tensioning the belts, X had moved to 50.6 Hz, but the config was still tuned for 54 Hz.

| Axis | Before | After |
|---|---|---|
| X | `mzv` @ 54.0 Hz, damping 0.093 | `ei` @ 52.2 Hz, damping 0.139 |
| Y | `mzv` @ 33.4 Hz, damping 0.079 | `mzv` @ 33.8 Hz, damping 0.082 |

- Y keeps clean acceleration to about **3,350 mm/s²**; X allows about 5,160.
- Slicer accelerations checked against this (OrcaSlicer, unchanged):
  - outer wall and top surface: 3,000
  - default and inner wall: 5,000
  - travel: 7,000

### Belt re-tension and Shake&Tune (mechanical; no config change)
All tests used Shake&Tune v5.1.1. Graphs are in `~/printer_data/config/ShakeTune_results/` on the printer.

| Run | Belt similarity | Main belt peaks (A / B) | Y peak | X peak |
|---|---|---|---|---|
| Jul 2025 (earlier results) | 95.2%, "good health" | about 34 / 34 Hz and 56 / 59 Hz | — | — |
| 2026-10-13 06:42, before tensioning | 89.7% | about 35 / 32 Hz and 49 / 52 Hz | 30.1 Hz (+ 55 Hz) | 47.5 Hz |
| 07:27, first re-tension | 89.0% | 35 / 34 Hz and 52 / 58 Hz | — | — |
| 09:06, belt A (rear-right motor) tightened | 88.0%, "likely mechanical issue" | 35.0 / 36.2 Hz and 52 / 58 Hz | — | — |
| 09:15, axis test | — | — | **33.2 Hz, single clean peak** | **50.6 Hz** |

- Belt A = `stepper_y` (rear-right motor); belt B = `stepper_x` (rear-left motor).
- **Still open:** the 52 vs 58 Hz peak pair and a 110–133 Hz bump (about 30% stronger on belt B, also visible on X) didn't respond to tension. Suspects:
  - idlers or bearings on belt B's path
  - belt alignment, or a belt riding on a flange
  - X-carriage belt clamps or toolhead screws
  - motor pulley grub screws

## 2026-10-12

### Toolhead undervoltage shutdown (no config change)
- `TMC 'extruder' reports error: GSTAT: 00000004 uv_cp=1(Undervoltage!)`
- Happened during `Gridfinity_ABS_6h47m.gcode`, about 74% through, at layer Z 21.75.
- Conditions at the time:
  - bed 110°C and nozzle 255°C, both holding
  - part fan at 10%
  - CAN bus clean, no communication errors
- First occurrence in every log back to Dec 2024.
- **Likely cause:** a brief 24 V dip at the toolhead board (EBB). Check the toolhead power connector, the umbilical crimps at both ends, umbilical bend fatigue, and the PSU terminals.

### Hardware recorded (not in the config files)
- Filametrix Stealthburner, Clockwork 2 extruder
- E3D Revo 24 V, 1.75 mm, **60 W HeaterCore**, standard 0.4 mm nozzle
- Expected flow for ABS at 255°C was about 14–15 mm³/s, estimated from E3D's published data. Measured 19.5 mm³/s on 2026-10-13 (see the flow test above).

## 2026-10-10

### Adaptive Nevermore control — commit `5adb5ea`
- **New file:** `config/nevermore.cfg`, which replaces `config/chamber-fan.cfg`.
  - `chamber-fan.cfg` was deleted from the printer after a successful restart.
  - The `[temperature_sensor chamber_sensor]` section was removed; that temperature now shows on the `nevermore` row in Mainsail.
- **Why:**
  - Running the Nevermore too fast stopped the bed reaching temperature, which caused heater errors.
  - The old macros overrode `M140`, `SET_HEATER_TEMPERATURE` and `TURN_OFF_HEATERS`, and that turned the hotend off.
- **What it does:** `[temperature_fan nevermore]` on PD12, shown in Mainsail like the MCU fan.
  - Stays off while the bed heats, until it's within 3°C of target.
  - Then ramps up in 10% steps every 30 s, up to 100%, only while the bed heater has power to spare.
  - Backs off if the bed sags.
  - Runs only for bed targets of **90°C or more**.
  - After a hot print, scrubs at 100% until the bed is **below 45°C**.
  - Prints `Nevermore:` console messages, and warnings for bed sag, chamber at 70°C or more, and manual target changes.
  - Commands: `NEVERMORE_STATUS`, `NEVERMORE_SET SPEED=`, `NEVERMORE_AUTO ENABLE=`.
- `macros.cfg`: the heat soak now uses `_NEVERMORE_DWELL` instead of `G4`, so the fan loop keeps running during the soak.
- `voron-24-350.cfg`: `[include chamber-fan.cfg]` changed to `[include nevermore.cfg]`.
- `M190` is overridden so it waits for the bed with `TEMPERATURE_WAIT`.

## 2026-10-09

### Case light flicker workaround — commit `e9a73fa`
- `config/chamber-leds.cfg`: case light effects slowed to 8 fps with gentler animations.
- **Why:** the right-hand strip is spliced in parallel on the same data line and gets corrupted data.
- **Hardware fix still pending:** a 330–470 Ω series resistor on the right branch, a ground check or level shifter, or rewiring the strips in series with `chain_count: 42`.

### LED status effects — commit `7b93864`
- `config/chamber-leds.cfg`: [klipper-led_effect](https://github.com/julianschill/klipper-led_effect) effects for the case light (21 LEDs) and the Stealthburner (logo plus nozzle).
  - States: idle, heating, leveling, printing, paused, done, error.
  - The nozzle LEDs flash on every Tap probe.
- `_LED_STATE STATE=<state>` macro. "done" returns to idle after 5 minutes and "error" after 2 minutes.
- `macros.cfg`:
  - `PRINT_START`, `PRINT_END` and `G32` set the LED states.
  - Mainsail's pause, resume and cancel hooks are set through `_CLIENT_VARIABLE`.
- On the printer: `~/klipper/klippy/extras/led_effect.py` was symlinked to `~/klipper-led_effect/src/led_effect.py`. The installer needs sudo, so this was done by hand.

### Repo baseline — commits `e364a53`, `7a15178`
- Live config pulled from the printer (Klipper v0.13.0-154) as the baseline.
- `.gitattributes` keeps `config/**` on LF line endings to match the printer.

---

## Open items
- [ ] Track down the belt B / X-axis 110–133 Hz resonance and the belt peak mismatch, then re-run Shake&Tune.
- [ ] Inspect toolhead 24 V wiring after the undervoltage shutdown.
- [ ] Case light right-strip wiring fix, then revert the slowdown.
- [x] Orca flow test at 255°C: limit 19.5 mm³/s (2026-10-13). Raise the filament cap to 16 and the inner wall and infill speeds in Orca.
- [ ] Recalibrate pressure advance (currently 0.02).
- [ ] Config review items:
  - `max_accel: 44525` → set a realistic limit
  - `verify_heater` limits loosened (bed `max_error: 500`)
  - `M141` called but undefined
  - stray `"0` typo in the `PRINT_START` "complete" message
