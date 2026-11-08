# Tevo Tornado: SKR 1.4 Turbo → MKS Monster8 V2 Migration Plan

> Status: **planning / reference**. Nothing in `printer_data/config/` has been changed yet.
> Source of truth for pins: Makerbase official pin diagram `MKS Monster8 V2.0_003 PIN.pdf`
> (github.com/makerbase-mks/MKS-Monster8, `hardware/MKS Monster8 V2.0_003/`), cross-checked
> against Klipper's `config/generic-mks-monster8.cfg` and MKS's `printer_v2.cfg`.

---

## 1. The board at a glance

| Feature | MKS Monster8 V2 | Old SKR 1.4 Turbo | What it means for the Tevo |
|---|---|---|---|
| MCU | STM32F407VET6, 168 MHz, 512 KB flash, 192 KB RAM | LPC1769, 120 MHz | Faster, more step-rate headroom; well supported by Klipper |
| Stepper drivers | **8 sockets** (9 motor connectors: Driver2 has two parallel plugs Z1/Z2) | 5 | 3 spare drivers after this build |
| Driver modes | UART or SPI per socket (jumpers), 3.3 V / 5 V driver logic jumper | UART | Reuse your 5× TMC2209 in UART mode |
| Heater outputs | 4: H-BED (PB10), HE0 (PB1), HE1 (PB0), HE2 (PA3) | 3 | 2 spare MOSFETs (VIN voltage) |
| Thermistors | 4: TB (PC0), TH0 (PC1), TH1 (PC2), TH2 (PC3) | 4 | 2 spare (chamber / PSU temp) |
| PT100/PT1000 | On-board MAX31865 (uses Driver7/E4 pins) | none | High-temp upgrade path, costs Driver7 |
| PWM fans | 3: FAN0 (PA2), FAN1 (PA1), FAN2 (PA0), each jumper-selectable 5 V / 12 V / VIN | 3 + heater-as-fan | One fewer true fan port than you use today — see §4 |
| Always-on power outputs | 3, jumper-selectable 5 V / 12 V / VIN | — | Good for fans that just need to run |
| Endstop/IO ports | X- (PA14), Y- (PA15), Z- (PB13), Z+ (PB12), MT_DET (PA13) | 6 + PWRDET | Z+ power jumper selectable **5 V or VIN** |
| Dedicated headers | 3D Touch servo (PA8), Neopixel (PC5), EXP1/EXP2, UART1 (PA9/PA10), I2C, SPI3, SWD | — | BLTouch + LED ring have native homes |
| CAN | **Built-in transceiver** + RJ11 port, CAN1 on PB8/PB9 (shared with I2C EEPROM via jumper) | none | Enables USB→CAN bridge + a toolhead board |
| Power | 12–24 V in, onboard 12 V + 5 V bucks, **5 V 3 A out for a Pi**, reverse-polarity + TVS protection, back-EMF protection on motor outputs | — | More robust than the SKR 1.4 |
| Storage / flashing | TF card, U-disk (USB-A), DFU via BOOT0 button | SD | SD-card flashing (no `make flash`) |

### Shared signals you must know about
| Shared item | Effect | Action |
|---|---|---|
| X and Y **share an enable pin** (PC15) | `M84`/`SET_STEPPER_ENABLE` on X also drops Y | Harmless for a cartesian; just don't be surprised |
| E0 (Driver3) and E4 (Driver7) **share an enable pin** (PB6) | Whatever is on Driver7 enables/disables with the extruder | **Never put a Z motor on Driver7** |
| Driver2 has **two motor plugs on one driver** (Z1 and Z2 sockets) | Both plugs move identically | Do **not** use for your dual-Z — Z_TILT needs independent drivers |
| DIAG jumpers link driver DIAG pins → X-, Y-, Z-, Z+, MT_DET, Neopixel | A fitted jumper lets the driver yank your endstop/LED pin | **Remove all DIAG jumpers** (you use physical switches) |
| CAN/I2C jumper (PB8/PB9) | Default = I2C EEPROM | Only move to CAN if you go the toolhead-board route |

---

## 2. Hardware inventory being carried over (from current `printer.cfg`)

| Function | Current SKR pin | Notes |
|---|---|---|
| X stepper | Driver X, uart P1.10 | 0.90 A, mechanical endstop `^!P1.29` |
| Y stepper | Driver Y, uart P1.9 | 1.10 A, mechanical endstop `^!P1.28` |
| Z left | Driver Z, uart P1.8 | Independent, Z_TILT |
| Z right | Driver E1, uart P1.1 | Independent, Z_TILT |
| Extruder (BIQU H2 V2S) | Driver E0, uart P1.4 | 0.80 A |
| Hotend heater / thermistor | P2.7 / P0.24 | 65 W, Generic 3950 |
| Bed heater / thermistor | P2.5 / P0.25 | **24 V DC bed via external MOSFET module** (confirmed) |
| BLTouch clone | sensor ^P0.10, control P2.0 | |
| Part cooling (dual 5015) | P2.3 | |
| Hotend heatsink fan | P2.4 (HE1 used as fan) | |
| Electronics fan (temp-controlled) | P1.27 + thermistor P0.23 | |
| PSU fan 1 / PSU fan 2 | P1.31 / P3.25 | Manual on/off |
| Neopixel ring (16 LED) | P1.24 | |
| SFS v2 runout switch | ^P1.25 | |
| SFS v2 motion encoder | ^P1.26 | Currently disabled |
| Power-loss detect | ^P1.0 (PWRDET) | **No device fitted** (confirmed) — `[gcode_button power_loss_detect]` will be dropped |
| LIS2DW accelerometers ×2 | USB RP2040 (separate MCUs) | Unaffected by board swap |

---

## 3. Proposed pin map (baseline, no CAN toolhead)

| Function | Monster8 V2 port | Pin(s) | Jumper / wiring notes |
|---|---|---|---|
| X | Driver0 | STEP PC14, DIR PC13, EN !PC15, UART PE6 | UART jumper set |
| Y | Driver1 | STEP PE5, DIR PE4, EN !PC15, UART PE3 | UART jumper set |
| Z left (`stepper_z`) | Driver2 — **Z1 plug only** | STEP PE1, DIR PE0, EN !PE2, UART PB7 | Leave Z2 plug empty |
| Extruder | Driver3 (E0) | STEP PB5, DIR PB4, EN !PB6, UART PB3 | |
| Z right (`stepper_z1`) | Driver4 (E1) | STEP PD6, DIR PD5, EN !PD7, UART PD4 | Independent driver for Z_TILT |
| *spare* | Driver5 (E2) | STEP PD2, DIR PD1, EN !PD3, UART PD0 | |
| *spare* | Driver6 (E3) | STEP PC7, DIR PC6, EN !PC8, UART PD15 | |
| *spare (shares EN with extruder)* | Driver7 (E4) | STEP PD13, DIR PD12, EN !PB6, UART PD11 | Also the MAX31865 pins |
| Hotend heater | HE0 | PB1 | |
| Hotend thermistor | TH0 | PC1 | |
| Bed MOSFET module signal | H-BED | PB10 | Board output drives the external MOSFET module's signal input; bed current stays off the board |
| Bed thermistor | TB | PC0 | |
| Part cooling | FAN0 | PA2 | Fan voltage jumper → match fans (VIN for 24 V) |
| Hotend heatsink fan | FAN1 | PA1 | Same |
| Electronics fan | FAN2 | PA0 | Driven by MCU temp (no extra thermistor needed) |
| PSU fan 1 / 2 | HE1 / HE2 | PB0 / PA3 | VIN (24 V) — matches your 24 V fans; controlled, not always-on |
| X endstop | X- | PA14 | |
| Y endstop | Y- | PA15 | |
| BLTouch probe signal | Z- | PB13 | 5 V-only port, perfect for BLTouch |
| BLTouch servo/control | 3D Touch header | PA8 | |
| SFS runout switch | MT_DET | PA13 | 5 V port |
| SFS motion encoder | Z+ | PB12 | **Z+ power jumper MUST be on 5 V** |
| Neopixel ring | Neopixel header | PC5 | 5 V supply — check ring current (16 LEDs at your 25–30 % = fine) |
| Spare thermistors | TH1 / TH2 | PC2 / PC3 | Chamber temp, PSU temp |

---

## 4. Suggestions to use what the board can do

Ranked by value-for-effort. Items marked **(decision)** need your call before configs get written.

| # | Upgrade | Value | Cost / risk | My take |
|---|---|---|---|---|
| 1 | **Use the MCU's internal temp sensor** (`temperature_mcu`) for the electronics fan | Removes a thermistor + wire | None | Do it |
| 2 | **Controlled PSU fans** on HE1/HE2: thermostatic (thermistor in PSU on TH2) or `controller_fan` (on while steppers/heaters active) | Fans only run when needed; quieter | Fans are 24 V — compatible | Do it — method still to pick **(decision)** |
| 3 | **Chamber/ambient sensor on TH1** | Useful data, nice for ABS/ASA | One thermistor | Optional |
| 4 | **CAN toolhead board** (e.g. BTT EBB36 / EBB SB2209, MKS THR36) using the Monster8 as a USB→CAN bridge | 4-wire umbilical; toolhead accelerometer could replace the `lis_hotend` RP2040 | Extra purchase; more firmware steps | **Not now** (no CAN board owned) — future upgrade path; CAN/I2C jumper stays on I2C |
| 5 | **32 microsteps** with `interpolate: True` | Smoother/quieter motion; F407 has headroom | Halves max speed headroom vs 16; verify no "Timer too close" | Bring-up on 16, test 32 later |
| 6 | Katapult bootloader | Future flashes over USB/CAN, no SD-card shuffling | Overwrites MKS stock bootloader via DFU; small brick risk | Optional, after the board is proven |
| 7 | Sensorless homing (DIAG jumpers) | Fewer switches | Less repeatable on a heavy Y bed; conflicts with mechanical switches on the same pins | **Skip** — keep mechanical endstops |
| 8 | MAX31865 / PT1000 hotend sensor | > 300 °C capability | Burns Driver7 | Skip unless you go high-temp |
| 9 | Power the Pi from the board's 5 V 3 A rail | One less PSU | Pi 4 + KlipperScreen can exceed 3 A → undervoltage throttling | **Not used** — both Pi 4Bs (Klipper + webcam) already have proper supplies |

---

## 5. Setup strategy (phased — do not skip ahead)

### Phase 0 — Decisions (answered)
| Question | Answer | Effect on plan |
|---|---|---|
| CAN toolhead board? | No | Plain USB firmware; CAN/I2C jumper stays on I2C |
| Fan voltage | All 24 V | Every fan-voltage jumper → **VIN**; heater outputs usable for fans |
| Fan control preference | Controlled, not always-on | Always-on outputs unused; PSU fans on HE1/HE2 under Klipper control |
| Power-loss device | None | Drop `[gcode_button power_loss_detect]` |
| Pi | 2× Pi 4B (Klipper + webcam), own PSUs | Board 5 V Pi output unused |
| Bed | 24 V DC via external MOSFET module | H-BED output drives the module signal input |
| **Open:** PSU fan control method | — | Thermostatic (thermistor in PSU on TH2) vs. "on while printer is active" (`controller_fan`) |

### Phase 1 — Bench prep (board out of the printer)
| Step | Setting | Why |
|---|---|---|
| 1 | Driver mode jumpers → **UART** on each used socket | TMC2209 talks over UART |
| 2 | Driver logic voltage jumper → **3.3 V** | STM32 is 3.3 V logic |
| 3 | **Remove all DIAG jumpers** | They tie driver DIAG to X-/Y-/Z-/Z+/MT_DET/Neopixel |
| 4 | Fan voltage jumpers → match each fan (VIN for 24 V) | Wrong jumper = cooked fan |
| 5 | Z+ power jumper → **5 V** | SFS motion sensor is 5 V |
| 6 | CAN/I2C jumper → leave **I2C** unless doing CAN | |
| 7 | Power with 24 V, **no drivers/motors**, check 5 V and 12 V rails | Catches a bad buck before it eats parts |

### Phase 2 — Firmware
`make menuconfig` on the Pi:
- Enable extra low-level options
- Architecture: STMicroelectronics STM32 → STM32F407
- Bootloader offset: **48KiB bootloader (MKS Robin Nano V3)**
- Clock reference: **8 MHz crystal**
- Communication: **USB (on PA11/PA12)**  — *or* "USB to CAN bus bridge" with CAN on **PB8/PB9** if doing the toolhead board

Then `make`, copy `out/klipper.bin` to a FAT32 SD card as **`mks_monster8.bin`** (Klipper upstream's documented name; MKS's own README says `firmware.bin` — if one doesn't take, try the other). Insert, power-cycle. Fallback: DFU via BOOT0 button.
Verify with `ls /dev/serial/by-id/` → expect `usb-Klipper_stm32f407xx_<id>-if00`.

### Phase 3 — Config rewrite
- New `[mcu]` serial; drop `restart_method: command` (LPC-specific habit).
- Re-pin `printer.cfg`, `fans.cfg`, `sfs.cfg`, `neopixel_ring_unified.cfg`; remove the power-loss `gcode_button`.
- Macros (`core_macros.cfg`, `mainsail_include.cfg`, etc.) are pin-free and carry over unchanged.
- Clear the `SAVE_CONFIG` block values that no longer hold (see Phase 5).

### Phase 4 — Wiring
- Mechanical 2-wire switches go on **S and GND**, never across 5 V and GND (instant short of the 5 V rail).
- Label every cable at both ends while the machine is apart.
- Ferrules on all screw-terminal wires; torque the power terminals.

### Phase 5 — Commissioning checklist
| Check | Command | Pass criteria |
|---|---|---|
| Thermistors | Dashboard | Room temp on all sensors, no `ADC out of range` |
| TMC comms | `DUMP_TMC STEPPER=stepper_x` (each) | Registers read, no UART errors |
| Motor direction | `STEPPER_BUZZ STEPPER=…` (each) | Moves +1 mm then back; Z1/Z both raise gantry |
| Endstops | `QUERY_ENDSTOPS` pressed / released | open ↔ TRIGGERED flips correctly |
| BLTouch | `BLTOUCH_DEBUG COMMAND=pin_down/pin_up`, `QUERY_PROBE` | Deploys, retracts, triggers |
| Heater PIDs | `Extruder_PID`, `Bed_PID` macros | New values saved — different MOSFETs, re-tune |
| Probe offset | `Calibrate_Probe` | Re-do after rebuild |
| Z tilt | `BED_LEVEL` | Converges within `retry_tolerance` |
| Input shaper | `Shake` (with `s2dw.cfg` included) | Re-run — mechanics were rebuilt |
| Pressure advance | Tower test | Re-tune if extruder path changed |

---

## 6. Risks (explicit)

| Risk | Severity | Mitigation |
|---|---|---|
| DIAG jumpers left fitted | High — phantom endstop triggers, LED/sensor weirdness | Remove all of them in Phase 1 |
| Z+ jumper on VIN with SFS on it | High — 24 V into a 5 V sensor | Set 5 V before connecting |
| Fan jumper voltage mismatch | Medium — dead fans | Verify each fan's rating |
| Z motor placed on Driver7 | Medium — shares EN with extruder | Map per §3 |
| Using Driver2's second plug for Z right | High — breaks Z_TILT | Use Driver4 for Z right |
| SD flash filename ambiguity | Low | Try `mks_monster8.bin`, then `firmware.bin`, then DFU |
| Community CAN guides for this board are inconsistent | Medium (CAN route only) | Follow Klipper's official CANBUS docs; PB8/PB9 only |
| MKS QC variance | Low–Medium | Bench test in Phase 1 before installing |
