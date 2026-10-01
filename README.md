# Raspberry Pi 5 Troubleshooting

## Table of Contents

1. [Light is Red, Doesn't Boot](#1-light-is-red-doesnt-boot)
   - [1.1 Symptom](#11-symptom)
   - [1.2 What the LED Means](#12-what-the-led-means)
   - [1.3 First Isolation: Swap Test](#13-first-isolation-swap-test)
   - [1.4 Test A: Hard Drain and Minimal Config](#14-test-a-hard-drain-and-minimal-config)
   - [1.5 Test B: Visual Inspection](#15-test-b-visual-inspection)
   - [1.6 Test C: 5V Input Rail Resistance](#16-test-c-5v-input-rail-resistance)
   - [1.7 Test D: 3.3V Rail Resistance](#17-test-d-33v-rail-resistance)
   - [1.8 Test E: Bench Supply Current Draw](#18-test-e-bench-supply-current-draw)
   - [1.9 Short Localization](#19-short-localization)
   - [1.10 Decision Table](#110-decision-table)
   - [1.11 Rules and Notes](#111-rules-and-notes)

---

## 1. Light is Red, Doesn't Boot

### 1.1 Symptom

Plug in USB-C, the power LED stays solid red. Pressing the power button does nothing. The board never reaches green, never boots.

### 1.2 What the LED Means

On the Pi 5 the power LED is driven by the PMIC (DA9091):

- **Red, steady**: standby. Board has input power, the SoC has not started.
- **Green**: firmware is running, boot has begun.
- **Red staying red, button dead**: the board has standby rail but is not transitioning to boot. The power button is a soft button serviced by the PMIC, so an unresponsive button points at the power path, not the OS.

No green at all means the fault is below the firmware layer. A corrupt bootloader EEPROM would still give green with blink codes, so no green rules that out.

### 1.3 First Isolation: Swap Test

Before touching the board, split external causes from board causes. Move the suspect board's **PSU, cable, and SD card** onto a known-good Pi one at a time, or move a known-good set onto the suspect board.

- Known-good set boots on the suspect board → external cause (PSU, cable, card).
- Suspect card/PSU/cable boots a good board, but the suspect board stays dead on a good set → board-level fault.

Note: the SD card holds the OS, not the bootloader. The bootloader lives in the board's onboard SPI EEPROM. A card swap tells you nothing about EEPROM state.

Also confirm the PSU is a true 5V / 5A supply. A low-RAM board draws less and can boot on a marginal supply that fails a higher-draw board. Does not match the red-standby symptom (a weak supply browns out mid-boot instead), but it is a one-glance check.

### 1.4 Test A: Hard Drain and Minimal Config

Clears a PMIC that latched into a protective state on an overcurrent or thermal event.

1. Unplug everything.
2. Hold the power button ~10s to bleed residual charge.
3. Replug with no SD card and no peripherals.
4. Watch the LED.

- Goes green and waits for boot media → it was a latch, not a death.
- Stays red → not a recoverable latch, continue.

### 1.5 Test B: Visual Inspection

Inspect the PMIC and the inductors around the USB-C input, both sides of the board.

Look for discoloration, burnt smell, cracked or lifted components, bulged or split capacitors. Clean board does not clear it (most power failures leave no visible mark), but visible damage confirms it.

### 1.6 Test C: 5V Input Rail Resistance

**Dead board. USB-C unplugged, SD card out, nothing connected.** Resistance measurements are always done unpowered.

DMM in ohms mode, probe:

```
GPIO pin 2 (5V)  to  GPIO pin 6 (GND)
```

(Equivalent: USB-C VBUS pad to any ground.)

Expected:

| Reading | Meaning |
|---|---|
| ~13-14 kΩ (reference, known-good) | Normal. 5V input rail not shorted, not open. |
| A few Ω, near 0 | Shorted input rail. Dead PMIC or shorted bulk cap. Root cause found. |
| Open / megohms | Rail not shorted. Points at PMIC not sequencing, or SoC not starting. |

A normal reading here does **not** clear the board. A downstream short does not show up on the 5V input rail. Continue to Test D.

### 1.7 Test D: 3.3V Rail Resistance

**Dead board, same conditions as Test C.** This is the decisive test. A shorted downstream rail makes the PMIC refuse to sequence as a protective response, which produces exactly the red-standby, normal-5V-rail picture.

DMM in ohms mode, probe:

```
GPIO pin 1 (3V3)  to  GPIO pin 6 (GND)
```

Compare against a known-good board.

| Reading | Meaning |
|---|---|
| ~24 kΩ (reference, known-good) | 3.3V rail clean. |
| Large discrepancy vs good board, e.g. ~400 Ω | 3.3V rail shorted or partially shorted. This is the fault. |

Interpreting a partial short (e.g. ~393 Ω vs ~24 kΩ good):

- Not a dead 0 Ω short, so the PMIC output FET is not fused closed.
- A resistive pull-down on the rail. Most likely a failed decoupling ceramic (MLCC) gone leaky or partially shorted, a classic post-thermal-cycle failure. Less likely, a 3.3V peripheral IC failed internally.
- Prognosis shifts from "scrap" to "potentially repairable," because a shorted cap is findable and removable.

If D matches the good board, all accessible rails are clean and the fault is PMIC sequencing or SoC internal. Go to Test E. If D shows a short, skip E (the PMIC will hold off and E tells you nothing new) and go to Short Localization.

### 1.8 Test E: Bench Supply Current Draw

Run only if Tests C and D are both clean. Confirms a non-shorted PMIC/SoC failure.

The board is USB-C powered, so inject 5V by one of two routes. **One power source at a time. Wall PSU stays unplugged.**

**Route A, USB-C breakout (non-invasive).**
Use a USB-C breakout or PD trigger board that exposes VBUS and GND, run a USB-C cable to the Pi, feed supply + to VBUS, - to GND. Or cut a USB-C cable: VBUS is red, GND is black, verify with the DMM first. No PD negotiation off a dumb source, so current caps at 3A, irrelevant for a no-load test.

**Route B, GPIO 5V back-feed (fastest, splits the fault location).**
Feed supply into GPIO pin 2 or 4 (+5V), return on pin 6 (GND). Pin 1 is the square-pad corner by the J8 marking. Bypasses the USB-C connector and its input protection, drives the 5V rail straight to the PMIC.

Settings for either route: **CV 5.1 V, current limit 3 A.** Power on, read current off the supply's own ammeter.

| Result | Meaning |
|---|---|
| Near 0 A (tens of mA), LED stays red | PMIC powered but not enabling core rails. Dead PMIC, non-shorted mode. |
| Snaps to CC, voltage collapses | Shorted rail downstream (should have shown in C or D). |
| Inrush then settle ~0.3-0.5 A, LED green | Board is alive. Look elsewhere. |

Route B diagnostic split: if GPIO feed boots green but USB-C never does, the fault is isolated to the USB-C input front end (input FET, PD controller, protection) and the core board is fine. If GPIO also stays dead, it is PMIC core or SoC.

### 1.9 Short Localization

When Test D shows a shorted rail, find the offending component.

1. **Low-current injection, thermal trace.** Feed the shorted rail from the bench supply at low voltage (0.5-1 V), current limit ~0.5 A, injected across the rail (pin 1 to pin 6 for 3.3V). The short sinks current and the faulty part heats. Find the hot spot by touch, thermal camera, freeze spray, or isopropyl evaporation (wet the area, the hot part dries first). Keep injected voltage low and current-limited: the supply is a controlled current source to heat the fault, not to energize the rail normally.

2. **Visual on the rail's decoupling caps.** The cluster of small ceramics around the PMIC and SoC. Look for cracked, discolored, or lifted caps. A cracked MLCC is the classic post-thermal short.

3. **Confirm and remove.** Lift the suspect cap with hot air or roll it off with the iron. Re-measure the rail. If it jumps back toward the good-board value, that cap was the short. The board will bench-boot fine missing one decoupling cap, good enough to confirm. Replace the cap afterward.

### 1.10 Decision Table

| 5V rail (C) | 3.3V rail (D) | Bench current (E) | Verdict |
|---|---|---|---|
| Short (~0 Ω) | - | - | Shorted input. Dead PMIC or bulk cap. |
| Normal | Short (low Ω) | skip | Downstream short. Localize and remove (1.9). Repairable. |
| Normal | Normal | ~0 A, red | Dead PMIC, non-shorted. RMA or scrap. |
| Normal | Normal | green via GPIO, dead via USB-C | USB-C input front end fault. Core board fine. |
| Normal | Normal | green, ~0.3-0.5 A | Board alive. Re-check externals. |

### 1.11 Rules and Notes

- **Resistance tests are always dead-board.** USB-C unplugged, card out, nothing connected. Powering a board during a resistance measurement gives garbage readings and can damage the meter.
- **One power source at a time, ever.** Bench supply or wall PSU, never both.
- **85C is not a thermal kill.** Pi 5 silicon is rated well above that. Throttle starts ~80-85C, thermal shutdown is ~90C+ junction. A clean shutdown at a monitored 85C threshold is the monitor doing its job, not damage. A board that dies after such an event is usually coincidental timing pointing at a marginal PMIC, not heat destruction.
- **Reference values are from a known-good Pi 5 (8GB/2GB both read the same on these rails).** Always compare a suspect board against a good one rather than trusting an absolute number, since rail loading varies by model and revision.

GPIO power pins used above:

```
Pin 1  = 3V3
Pin 2  = 5V
Pin 4  = 5V
Pin 6  = GND
```
