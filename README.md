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
   - [1.9 Test F: rpiboot Enumeration](#19-test-f-rpiboot-enumeration)
   - [1.10 Short Localization](#110-short-localization)
   - [1.11 Decision Table](#111-decision-table)
   - [1.12 Rules and Notes](#112-rules-and-notes)
2. [Non-Destructive Diagnostic Path](#2-non-destructive-diagnostic-path)
   - [2.1 Read the 393 Ω Again: It Probably Is Not What Stops the Boot](#21-read-the-393-%cf%89-again-it-probably-is-not-what-stops-the-boot)
   - [2.2 Validate the Measurement](#22-validate-the-measurement)
   - [2.3 Rail Survey: Is 3V3 the Only Affected Rail?](#23-rail-survey-is-3v3-the-only-affected-rail)
   - [2.4 Powered Observations](#24-powered-observations)
   - [2.5 Decision Table and Stopping Rule](#25-decision-table-and-stopping-rule)
3. [Finding the Shorted Capacitor on the 3.3V Rail](#3-finding-the-shorted-capacitor-on-the-33v-rail)
   - [3.1 Read the Number Before You Reach for the Iron](#31-read-the-number-before-you-reach-for-the-iron)
   - [3.2 Step 0: Validate the 393 Ω](#32-step-0-validate-the-393-%cf%89)
   - [3.3 Build the 3V3 Cap Map from the Good Board](#33-build-the-3v3-cap-map-from-the-good-board)
   - [3.4 Narrow It Down Without Removing Anything](#34-narrow-it-down-without-removing-anything)
   - [3.5 Sequential Removal](#35-sequential-removal)
   - [3.6 After the Fix, and the Honest Prognosis](#36-after-the-fix-and-the-honest-prognosis)
   - [3.7 Tools](#37-tools)

Figure and diagram prompts: [PROMPTS.md](PROMPTS.md)

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

**Attach HDMI for this test.** With no boot media present, a healthy Pi 5 puts a boot diagnostic
screen on HDMI. That turns a negative test into a positive one:

- Diagnostic screen appears → firmware is running. The power path is fine and the fault is in boot
  media or the bootloader EEPROM, not the rails. Nothing in sections 2 or 3 applies.
- No screen, no green → the fault is below the firmware layer. Continue.

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

### 1.9 Test F: rpiboot Enumeration

Free, non-invasive, needs no bench supply, and it is the only test here that interrogates the SoC
directly instead of inferring its state from an LED. Run it before any rework.

The Pi 5's USB-C port carries USB 2.0 data as well as power, and the Pi 5 is a supported `rpiboot`
target. In device mode the SoC's boot ROM enumerates on a host PC before any boot media is read.

On a Linux host:

```
# host side, watch while you connect
dmesg -w
# or
watch -n1 lsusb
```

1. USB-C disconnected from the Pi.
2. Hold the Pi's power button down and keep holding.
3. Connect USB-C from the host PC to the Pi.
4. Watch for a Broadcom vendor ID appearing on the host.

| Result | Meaning |
|---|---|
| Enumerates (Broadcom VID appears) | SoC is alive and the PMIC sequences far enough to run the boot ROM. A shorted 3.3V rail is **not** what is stopping the boot. Re-examine the premise before opening section 3. |
| Does not enumerate | Consistent with the PMIC not sequencing. Confirms nothing new, but costs five minutes. |

Caveat: power and data share the one port. A host port that cannot supply enough current will fail
to enumerate a healthy board. Use a PD-capable port or a powered hub before trusting a negative
result.

### 1.10 Short Localization

When Test D shows a shorted rail, find the offending component.

The method depends on how hard the short is. Below ~20 Ω there is enough power at the fault to find it
thermally, and steps 1-3 apply. For a partial short of a few hundred ohms, as measured here, there is not,
and no thermal or current-injection method will work. Go to Section 2, then Section 3.

1. **Low-current injection, thermal trace.** *Hard shorts only, under ~20 Ω.* Feed the shorted rail from the bench supply at low voltage (0.5-1 V), current limit ~0.5 A, injected across the rail (pin 1 to pin 6 for 3.3V). The short sinks current and the faulty part heats. Find the hot spot by touch, thermal camera, freeze spray, or isopropyl evaporation (wet the area, the hot part dries first). Keep injected voltage low and current-limited: the supply is a controlled current source to heat the fault, not to energize the rail normally.

2. **Visual on the rail's decoupling caps.** The cluster of small ceramics around the PMIC and SoC. Look for cracked, discolored, or lifted caps. A cracked MLCC is the classic post-thermal short.

3. **Confirm and remove.** Lift the suspect cap with hot air or roll it off with the iron. Re-measure the rail. If it jumps back toward the good-board value, that cap was the short. The board will bench-boot fine missing one decoupling cap, good enough to confirm. Replace the cap afterward.

### 1.11 Decision Table

| 5V rail (C) | 3.3V rail (D) | Bench current (E) | Verdict |
|---|---|---|---|
| Short (~0 Ω) | - | - | Shorted input. Dead PMIC or bulk cap. |
| Normal | Hard short (< 20 Ω) | skip | Downstream short. Localize and remove per 1.10. Repairable. |
| Normal | Partial short (hundreds of Ω) | skip | Not necessarily the fault. Go to Section 2 before Section 3 — see 2.1. |
| Normal | Normal | ~0 A, red | Dead PMIC, non-shorted. RMA or scrap. |
| Normal | Normal | green via GPIO, dead via USB-C | USB-C input front end fault. Core board fine. |
| Normal | Normal | green, ~0.3-0.5 A | Board alive. Re-check externals. |

Test F cuts across the whole table:

| Test F (rpiboot) | Verdict |
|---|---|
| Enumerates on host | SoC alive, PMIC sequences. Whatever the rails read, the no-boot cause is above the power path. |
| Does not enumerate | No new information. Continue by rail readings. |

### 1.12 Rules and Notes

- **Resistance tests are always dead-board.** USB-C unplugged, card out, nothing connected. Powering a board during a resistance measurement gives garbage readings and can damage the meter.
- **One power source at a time, ever.** Bench supply or wall PSU, never both.
- **85C is not a thermal kill.** Pi 5 silicon is rated well above that. Throttle starts ~80-85C, thermal shutdown is ~90C+ junction. A clean shutdown at a monitored 85C threshold is the monitor doing its job, not damage. A board that dies after such an event is usually coincidental timing pointing at a marginal PMIC, not heat destruction.
- **Reference values are from a known-good Pi 5 (8GB/2GB both read the same on these rails).** Always compare a suspect board against a good one rather than trusting an absolute number, since rail loading varies by model and revision.
- **Never bridge 3V3 and 5V.** Simultaneous contact across pin 1 and pin 2 — a slipped probe, a dropped lead, a tilted meter tip — is fatal to the board. Tests C and D have you probing both in sequence, often on your only known-good reference unit. Clip one lead to ground and move only one probe at a time.
- **The Pi 5 has no polyfuse.** Neither does the Pi 4 or the Pi Zero. The advice repeated throughout no-boot threads — "leave it unplugged for days and let the polyfuse recover, do not re-power it too soon or it re-blows" — is real advice for older models and a dead end here. There is nothing to recover. Do not lose a day to it.
- **A steady red LED means input power is adequate.** The power LED blinking or extinguishing is the undervoltage signal. Solid red rules out PSU and cable as the cause, which matches the swap test in 1.3.

GPIO power pins used above:

```
Pin 1  = 3V3     Pin 17 = 3V3
Pin 2  = 5V      Pin 4  = 5V
Pin 6  = GND     (also 9, 14, 20, 25, 30, 34, 39)
```

---

## 2. Non-Destructive Diagnostic Path

Entry condition: Test D (1.7) read ~393 Ω from GPIO pin 1 to pin 6, against ~24 kΩ on the known-good
board. Section 3 takes that straight to the soldering iron. This section does not, and should be
worked to the end first — because the number itself argues that the iron is the wrong tool.

Everything here needs a multimeter, the known-good board, and patience. Nothing here removes a
component.

### 2.1 Read the 393 Ω Again: It Probably Is Not What Stops the Boot

Section 3.1 computes what 393 Ω across 3.3 V actually means:

```
I = 3.3 / 393   = 8.4 mA
P = 3.3² / 393  = 28 mW
```

It uses that to prove thermal localization cannot work. There is a second corollary it never draws,
and it matters more.

**8.4 mA is not an overcurrent.** The Pi 5's 3V3 rail feeds SoC I/O, RP1, the SD interface, the SPI
flash and the whole GPIO header — the header alone is specified to deliver hundreds of milliamps to
a HAT. A PMIC that refused to sequence because something drew an extra 8.4 mA could not boot a bare
board. Protective hold-off is a response to a hard short: a collapsed rail, a fused output FET, amps
into a dead node. Not 28 mW.

So the reasoning in 1.7 — shorted downstream rail, PMIC refuses to sequence, red standby — is sound
at 0-20 Ω and does not carry at 393 Ω.

Three readings of the measurement, ordered by likelihood:

1. **The 393 Ω is not a fault at all.** Surface contamination, flux residue, a measurement artifact,
   or a leakage path local to the GPIO header. The no-boot cause is elsewhere and entirely untouched
   by cap work. Section 2.2 settles this for free.
2. **The 393 Ω is real, and is a symptom rather than the cause.** A damaged IC on the 3V3 rail leaks
   through a degraded input or ESD clamp, *and* holds a reset, enable or power-good line in a state
   that stops the sequence. The leak is a fingerprint of the damage, not the mechanism that blocks
   boot. Removing capacitors finds nothing, because the leak is in silicon.
3. **The 393 Ω is real and is the cause.** This requires the PMIC to be far more sensitive than the
   rail's own load budget implies. Least likely of the three.

Only reading 3 justifies the cap hunt in section 3, and it is the weakest of the three. Note that
section 3.1's own probability ranking already agrees: at 200 Ω - 2 kΩ it ranks the causes as
contamination first, damaged IC second, leaky or cracked cap third. Skipping ahead to capacitors is
not the shortcut it looks like — it is starting with the least likely cause and the most expensive
tool.

### 2.2 Validate the Measurement

Free, and it resolves reading 1 outright.

3V3 appears on GPIO pins 1 and 17. GND appears on pins 6, 9, 14, 20, 25, 30, 34 and 39. A genuine
rail-to-ground leak is on the copper plane and must read the same from every combination. A
localized fault — debris, a solder splash, a whisker, a bent pin under the header — will not.

Conditions first, both boards, same session, exactly as 3.2 specifies: same meter, same manual
range, same leads; both boards completely bare; discharged; and **60 seconds of settling** before
reading, because the meter's test current is charging the rail's bulk capacitance and a number read
at two seconds is meaningless.

Then build the matrix. Sixteen measurements per board, about ten minutes:

| | pin 6 | pin 9 | pin 14 | pin 20 | pin 25 | pin 30 | pin 34 | pin 39 |
|---|---|---|---|---|---|---|---|---|
| **pin 1** | 393 Ω | | | | | | | |
| **pin 17** | | | | | | | | |

| Result | Meaning |
|---|---|
| All sixteen ≈ 393 Ω | Genuine rail-to-ground leakage. Continue to 2.3. |
| Any combination reads ≈ 24 kΩ | The fault is local to the low-reading pin pair, not the rail. Inspect under and around those two pins at magnification. Clean, re-measure. Cheapest possible outcome, and it does happen. |
| pin 1 low but pin 17 normal, or vice versa | Same conclusion. The fault is on or under the header. A two-minute fix. |

Then the two discriminators from 3.2, which separate reading 2 from reading 3 above. Do not skip
them:

- **Polarity.** Measure pin 1 to pin 6, then swap the leads. Symmetric (393 / 393 Ω) means a
  resistive path — cap, contamination, or a carbonised track. Asymmetric (e.g. 393 Ω one way, 8 kΩ
  the other) means a semiconductor junction: a damaged IC, **not** a capacitor. That is reading 2,
  and section 3 cannot fix it.
- **Diode mode.** Both boards, both polarities. A junction-like reading the good board does not show
  is the same signal.

Then clean. IPA and a soft brush, both sides, with attention to the PMIC cluster, under the GPIO
header, around the USB-C, the SD slot lip, and the HDMI and CAM/DISP connectors. **Dry completely** —
compressed air, then 30 minutes, or 15 minutes at 50 °C. Wet IPA is itself slightly conductive and
will hand you a false low reading. Re-measure the full matrix afterwards.

### 2.3 Rail Survey: Is 3V3 the Only Affected Rail?

The highest-information test in this document, and it needs nothing but the meter and the two
boards. Section 1 never asks the question.

The DA9091 generates several rails — core (~0.8 V), DDR (~1.1 V), ~1.8 V and 3.3 V. Whether the
leakage is confined to 3V3 or spread across rails is what separates a downstream single-rail fault
(repairable in principle) from a PMIC-internal or shared-plane failure (not repairable at home).

You cannot reach most rails from the header, but the bottom side carries roughly 76 gold test
points, large and probe-friendly. `img/pi5_board_bottom.png` shows them with labels legible across
TP2-TP5, TP7-TP21, TP23-TP26, TP28-TP34, TP36, TP38-TP39, TP41, TP43-TP50 and TP53-TP76.

Method. **Good board first**, bare and unpowered, since a mistake there is free:

1. Probe every reachable TP to GND (pin 6). Record the resistance.
2. Probe every reachable TP to pin 1 (3V3). Any TP reading **< 1 Ω** is on the 3V3 net — mark it.
   Use the low-ohm range and read the number, not the continuity beeper; on a net with this much
   capacitance the beeper chirps on the charging transient of almost anything.
3. You now have two groups: TPs on the 3V3 net, and TPs with some other resistance-to-ground
   signature.

Then repeat exactly the same list on the suspect board and compare column to column.

```
| TP | On 3V3? (good) | R to GND, good | R to GND, suspect | Δ |
```

| Result | Meaning | Next |
|---|---|---|
| Only the 3V3-group TPs read low vs good | Single-rail downstream fault, confined to 3V3 | Reading 2 or 3 from 2.1. Continue to 2.4. |
| Several unrelated rails read low | PMIC internal, or a shared plane / via fault | **Stop.** Not home-repairable. Section 3 will not help. |
| No TP differs, only GPIO pin 1 does | Header-local fault that 2.2 missed | Back to 2.2, inspect the header |
| A non-3V3 rail reads low while the 3V3 group matches | The 393 Ω was attributed to the wrong net | Re-identify the net before anything else |

This pass is tedious. It is also the only test you have that can say "stop, this is the PMIC" before
you spend a weekend on 0201s.

### 2.4 Powered Observations

Resistance work is always dead-board. These three are deliberately powered, and measure volts and
amps rather than ohms. **Wall PSU only. One power source at a time, as ever.**

**(a) Is 3V3 actually coming up?** Official PSU connected, board sitting in its red-standby state.
DMM in DC volts, pin 1 to pin 6.

| Reading | Meaning |
|---|---|
| 0 V | The PMIC is holding the rail off. Consistent with a sequencing fault — though see 2.1 on why 8.4 mA is an unlikely trigger for one. |
| ~3.3 V, stable | **The rail is up**, regulating fine into the 393 Ω, exactly as 2.1 predicts it would. The leak is not blocking the boot. Stop chasing it; the fault is above the power path. |
| Sagged (1-2 V) or oscillating | The rail is being driven into a load it cannot hold. This is the one result that promotes the leak to prime suspect and justifies section 3. |

Run (a) before opening section 3. A stable 3.3 V here invalidates the entire cap hunt on its own.

**(b) PD contract and standby current.** An inline USB-C PD meter (~$15) between the official PSU
and the board gives both at a glance. It is the best single purchase in this exercise.

- **PD contract.** A Pi 5 on the 27 W supply should negotiate 5 V and request 5 A. No contract, or a
  fall back to the default 5 V / 3 A, points at the USB-C PD front end — the CC lines or the PD
  controller. That is an independent and well-known Pi 5 no-boot cause which produces exactly this
  symptom and has nothing to do with the 3V3 rail.
- **Standby draw.** Compare suspect against good, both at solid red. Near-identical and small means
  the PMIC is not attempting to sequence. Materially higher on the suspect means something
  downstream is enabled and sinking — which 8.4 mA would not explain.

**(c) 5 V present.** Pin 2 to pin 6, DC volts, powered. Confirms the input path independently of the
PD reading.

### 2.5 Decision Table and Stopping Rule

Work 2.2, 2.3, 2.4 and Test F (1.9) in that order, then read off:

| Finding | Verdict | Action |
|---|---|---|
| Matrix inconsistent, or rail recovers after cleaning | Contamination or header-local leak | Fixed. Re-run Tests C, D, then power up. |
| Polarity asymmetric / junction in diode mode | Damaged IC on the rail | Section 3 cannot fix this. Stop or RMA. |
| Multiple unrelated rails low (2.3) | PMIC internal or plane fault | Stop. Not home-repairable. |
| Test F enumerates | SoC alive, PMIC sequences | The 393 Ω is incidental. Chase bootloader EEPROM and boot media. |
| HDMI diagnostic screen appears with no media (1.4) | Firmware runs | As above. Power path is fine. |
| 3V3 measures a stable 3.3 V under power (2.4a) | Rail regulates into the leak | The leak is not the blocker. Look above the power path. |
| No PD contract (2.4b) | USB-C PD front end | Independent fault. Confirm with Test E route B (1.8). |
| Single rail, symmetric, survives cleaning, 3V3 stays at 0 V under power | Genuine 3V3 downstream fault | Only now does Section 3 make sense. |

**Stopping rule.** Only the last row justifies opening section 3. If you reach the end of this
section without landing on it, the answer is not a capacitor, and no amount of hot air will find
one. Set that rule before you start rather than discovering it four hours in.

---

## 3. Finding the Shorted Capacitor on the 3.3V Rail

**Entry condition: you worked Section 2 to the end and landed on its last row** — the leak is confined
to 3V3, symmetric under polarity reversal, survived cleaning, and 3V3 measures 0 V under power. If
you have not done that, go back. Section 2.1 explains why 393 Ω is unlikely to be what stops the
boot at all, and section 2.5 lists six cheaper findings that each rule this section out entirely.

Given that condition, this section takes the fault from "there is a short" to "this is the part."

You have a known-good Pi 5. That is worth more than a schematic here, and the method below is built around it.

Reference board images: `img/pi5_board_top.png` and `img/pi5_board_bottom.png` (clean reference Pi 5,
test-point labels legible), with `img/pi5_top_annotated.png`, `img/pi5_bottom_annotated.png`,
`img/pi5_sections.png` and `img/pi5_rail_topo.png` as region maps. These are **reference boards, not
the suspect unit** — use them for locating test points and planning the sweep, never as evidence
about this board's condition. The original close-up photographs of the suspect board were removed
from the repository in `9cce0aa`; re-shoot them before relying on any visual observation below.

### 3.1 Read the Number Before You Reach for the Iron

393 Ω across 3.3 V:

```
I = 3.3 / 393   = 8.4 mA
P = 3.3² / 393  = 28 mW
```

28 mW spread across a 0402 body or an IC die is nothing. Two consequences, and they are the whole reason this section exists:

- **Thermal localization will not work.** Nothing dissipating 28 mW gets measurably warm. Thermal camera, freeze spray, and the isopropyl-evaporation trick all need roughly 0.5 W at the fault, which means a short under ~20 Ω. Section 1.10 step 1 applies to a hard short, not to this one.
- **Voltage-gradient (µV drop) tracing will not work either.** It needs hundreds of mA pushed into the rail to raise a measurable IR gradient in the copper plane. At 393 Ω, 500 mA would take ~200 V. Not available, not survivable.

What is left is measurement validation, a cap map built from the good board, non-destructive discrimination, and sequential removal. In that order.

Resistance band to likely cause:

| Rail to GND | Most likely cause |
|---|---|
| 0 - 20 Ω | Solder bridge, conductive debris, fully shorted MLCC, fused PMIC output FET |
| 20 - 200 Ω | Cracked MLCC conducting through the crack |
| **200 Ω - 2 kΩ** | **Leaky MLCC, damaged IC input / ESD clamp, or surface contamination** |
| > 5 kΩ | Usually not a fault. Verify against the good board before chasing it |

393 Ω lands in the third band. A capacitor is a reasonable suspect there but it is not the favourite. Ranked by probability: surface contamination, damaged IC on the rail, leaky or cracked cap. Section 3.2 eliminates the first one cheaply, and tells you whether you are looking at silicon instead of a cap.

### 3.2 Step 0: Validate the 393 Ω

A large share of apparent partial shorts are measurement artifacts or dirt. Clear both before any rework.

Conditions, both boards, same session:

1. **Same meter, same range, same leads.** An autoranging meter uses different test currents on different ranges and will hand you two different numbers for the same net.
2. **Both boards fully bare.** No SD card, no PCIe FFC, no camera/display FFC, no fan, no RTC battery on J5, no HAT on J8, nothing in USB or HDMI, no PSU. An M.2 HAT with its own shorted cap is still a short on your 3V3 rail, and it is not on your board.
3. **Discharged.** Everything unplugged, hold the power button ~10 s, then briefly short pin 1 to pin 6 with a lead.
4. **Let it settle 60 s.** The meter's test current charges the rail's bulk capacitance, so the reading climbs for the first several seconds. A number read at 2 s is meaningless.

Then run these four checks.

**Polarity test.** Measure pin 1 to pin 6, then swap the leads.

| Result | Meaning |
|---|---|
| Same both ways (393 / 393 Ω) | Resistive path. Cap, contamination, or carbonised track. Continue in this section. |
| Asymmetric (e.g. 393 Ω / 8 kΩ) | Semiconductor junction. A damaged IC on the rail, **not** a cap. Jump to the fallback in 3.5. |

**Diode mode.** Both boards, both polarities, pin 1 to pin 6. Note the mV. A junction-like reading the good board does not show is the same signal as above.

**Second access point.** Repeat at GPIO pin 17, also 3V3. Same net, so it must match pin 1. If pin 1 reads low and pin 17 reads normal, the fault is on or under the header itself: bent pin, solder splash, or debris between pin 1 and the pin 6 / pin 9 grounds. That is a two-minute fix. Check it.

**Clean the board.** IPA and a soft brush, both sides, with attention to the PMIC cluster, under the GPIO header, around the USB-C, the SD slot lip, and the HDMI and CAM/DISP connectors. Dry it fully — compressed air, then 30 min, or 15 min at 50 °C. Re-measure. Flux residue, condensation salts and solder swarf cause a few-hundred-ohm leak routinely, and the fix is free.

Only a number that survives all four is worth cutting into.

Optional weak signal: measure capacitance pin 1 to pin 6 on both boards. A 393 Ω parallel path makes most DMM capacitance ranges read garbage or time out, so treat a wild difference as further confirmation of the resistive path, not as evidence against a specific cap.

### 3.3 Build the 3V3 Cap Map from the Good Board

Raspberry Pi does not publish a full Pi 5 schematic or a component designator map. You do not need one.

The 3V3 net touches dozens of 0201 and 0402 ceramics spread across the PMIC cluster, the SoC, RP1, the SPI flash area, the SD slot and the connectors. You cannot work on all of them, and most of the caps around the PMIC are on *other* rails — the DA9091 also makes the 0.8 V, 1.1 V and 1.8 V rails. Pulling one of those teaches you nothing and costs you a part.

So build the map first, on the **good** board, where a mistake is free.

Method, good board, unpowered and bare:

1. Clip or tape one probe to GPIO pin 1 (3V3). You need a free hand.
2. With the other probe, touch each pad of every small passive in the target regions. Use the **low-ohm range and read the number** — not the continuity beeper. On a net with this much capacitance the beeper chirps on the charging transient of almost any cap and gives constant false hits.
3. A pad reading **< 1 Ω** is on the 3V3 net. Record it.
4. Confirm the part's *other* pad reads < 1 Ω to GND (pin 6). Both conditions together mean it is a 3V3 decoupling cap. One pad on 3V3 and the other going somewhere else is a series element — leave it alone.

Regions to sweep, in order of how much your time is worth there:

| Region | Where | Note |
|---|---|---|
| PMIC cluster | Top side: the Renesas-marked IC beside the USB-C connector, with the inductor bank | Highest cap density on the board, and most of it is not 3V3. The map matters most here. |
| SPI flash / RTC | Bottom side, around the FLASH WP silkscreen, TP13 / TP14 / TP16 | Few parts, easy access, worth doing early |
| SD slot | Bottom side, around J9 and the SOT-23 beside it | 3V3 card power, easy access |
| Connector rails | Top side: CAM/DISP J3 / J4, PCIe J20, HDMI, PoE header | Easy access, low part count |
| RP1 | Bottom side, the cluster near TP64 / TP44 under the RP1 BGA | 3V3 I/O present, mixed with other rails |
| SoC decoupling | Bottom side, the dense 0201 field under the BCM2712 footprint | Mostly core and DDR rails. Hardest to rework. Last. |

Also sweep the bottom-side test points. Several TPs sit on power rails, and they are gold, large and probe-friendly. Any TP reading < 1 Ω to pin 1 on the good board gives you convenient 3V3 access on the suspect board without stabbing at 0201 pads.

Record as you go:

```
| # | Side | Region | Nearest marking | Size | On 3V3? | Notes |
```

Then photograph both boards at the same angle and annotate the map onto the photo. `img/pi5_board_top.png` and `img/pi5_board_bottom.png` work as layout bases, and `img/pi5_top_annotated.png` / `img/pi5_bottom_annotated.png` show the region breakdown already applied to them.

### 3.4 Narrow It Down Without Removing Anything

Work only on parts your map says are on 3V3. Three passes, cheapest first.

**Pass 1 — optical, 20-40×.** USB microscope or a loupe. Compare suspect against good, region by region, same magnification. Look for a diagonal crack across an MLCC body (the classic flex and thermal-cycle failure, usually near a board corner, a mounting hole, or a connector that gets levered), a chipped corner, darkening or a brown halo on the body or pads, a solder ball or whisker bridging a part, a lifted or tombstoned part. On this board also check the white residue previously recorded on the bottom side near TP64/TP10 and near TP61/TP32 — probably flux, but confirm it is not a leakage path before dismissing it. The close-up photograph that observation came from is no longer in the repository, so re-shoot that area and re-confirm rather than treating it as established. Note that 2.2's cleaning step may already have removed it. Photograph anything suspicious before touching it.

**Pass 2 — press test.** Meter on pin 1 / pin 6, eyes on the reading. With a wooden or plastic probe, never metal, press each mapped 3V3 cap firmly, one at a time. A cracked MLCC often changes resistance under pressure as the crack faces move. A jump of more than a few percent marks that part. No change anywhere does not clear the board.

**Pass 3 — localized heat.** The one technique that actually works at 393 Ω. Leakage through a cracked dielectric, and leakage through a damaged junction, both rise steeply with temperature, while healthy parts barely move.

1. Meter on pin 1 / pin 6, baseline logged. Let the board sit at room temperature 10 min first.
2. Fine hot-air nozzle (3-4 mm) at **low airflow, 150-200 °C**, or a clean iron tip at ~150 °C touched to the part body. You are warming, not reflowing.
3. Heat one mapped 3V3 cap ~10 s. Watch the meter.
4. Let it cool 30 s. Next part.

Interpretation: the faulty part drops the rail resistance noticeably and reversibly while hot — tens of percent, not single digits. Warm a few known-good parts first to learn what "nothing" looks like on your setup; that is your control.

Cautions. Keep airflow low or you will launch 0201s. Heat conducts a couple of millimetres, so a hit localizes a *neighbourhood*, not a part — confirm by re-heating each candidate individually from different directions. Do not exceed ~200 °C or you start reflowing joints.

This pass tells you **where**, not **what**. If the hot spot contains an IC as well as caps, the IC is still in play.

### 3.5 Sequential Removal

Only after 3.2, 3.3 and 3.4. If a pass flagged a part, pull that one first. Otherwise work the map in order of access and consequence:

1. Parts flagged by the press or heat test.
2. Visually suspect parts.
3. Easy-access, low-consequence regions: connector rails, SD slot, SPI flash area.
4. PMIC cluster 3V3 caps.
5. SoC and RP1 bottom-side 0201s. Last — highest risk of collateral damage, hardest to replace.

Rework setup for a Pi 5:

- **Preheat is not optional.** This is a multilayer board with heavy internal copper that sinks heat away fast. Bottom preheater or hot plate at 130-150 °C, soak 3-5 min. Without it you dwell on the joint until the pad lifts.
- Hot air: 4 mm nozzle, 330-360 °C, the lowest airflow that still works.
- Flux the part and add a little leaded solder to both ends first. The leaded mix drops the melting point and the part releases sooner.
- Kapton over neighbouring parts. A 0201 three millimetres away will take off.
- Photograph the area before every removal. 0201s are 0.6 × 0.3 mm and you will not remember where it came from.
- Each removed part into its own labelled piece of tape. You are putting them back.

The loop, per part:

```
1. Photograph the area
2. Remove the part
3. Measure the removed part off-board, ohms, both polarities
4. Measure the rail: GPIO pin 1 to pin 6, 60 s settle
5. Log it
```

| Removed part reads | Rail after removal | Verdict |
|---|---|---|
| Low Ω (hundreds or less) | Jumps toward ~24 kΩ | Found it. That part was the short. |
| Open (megohms) | Jumps toward ~24 kΩ | Re-measure both. Usually the removal cleared a bridge or debris rather than the part being bad. Inspect the pads. |
| Open | Still ~393 Ω | Not it. Reinstall or set aside, continue. |

Log template:

```
| # | Part / location | Removed reading | Rail after | Reinstalled? |
```

**Fallback: every mapped 3V3 cap is off and the rail still reads ~393 Ω.** Then the load is an IC on the rail — which the 3.2 polarity test has probably already told you. Candidates, ordered by how attackable they are: the SPI flash, the small RTC and ancillary parts, the SD interface parts, the Ethernet PHY, RP1, the SoC, the PMIC itself. Realistically only the small ones are worth removing. Pulling RP1, the BCM2712 or the PMIC to prove a diagnosis destroys the board's salvage value and needs a reball to undo. At that point take the call in 3.6.

### 3.6 After the Fix, and the Honest Prognosis

If the rail recovered:

- The board will bench-boot missing one decoupling cap. Do that first to confirm the fix: re-run Test C (1.6), Test D (1.7), then power up.
- Then replace the part. Get the value by measuring the equivalent part on the good board with an LCR meter, or by matching it to its identical neighbours.
- Do not leave a cap off permanently. Missing decoupling on a 3V3 rail feeding RP1 or the SoC shows up later as intermittent instability, which is a much worse fault to chase than this one.
- Re-measure pin 1 to pin 6 after reinstalling. It should sit at the good board's value.

Prognosis, stated plainly so you can decide where to stop:

- The cheap steps — 3.2 validation and cleaning — genuinely resolve a real share of few-hundred-ohm faults, and they cost about an hour.
- Past that you are hunting an unlabelled part on a board with no published schematic, mostly 0201s, mostly on the dense bottom side. With a microscope, hot air and preheat it is doable. Without all three it is not.
- A replacement Pi 5 costs less than the rework gear this needs. The reason to continue is that you want to, not that it pays. That is a fine reason — just do not discover it four hours in.

Set a stopping rule before you start. A reasonable one: stop after 3.4 if nothing is flagged and you do not already own preheat, hot air and magnification.

### 3.7 Tools

Required:

- DMM, 4+ digits, manual range, decent low-ohm accuracy. Both boards measured with this one meter.
- 20-40× magnification: USB microscope or stereo scope.
- Hot air station with fine nozzles, plus a preheater or hot plate.
- Fine tweezers, flux, leaded solder, braid, IPA, Kapton tape.
- The known-good Pi 5, same board revision where possible.

Useful:

- LCR meter, to read replacement values off the good board.
- Bench supply — for Test E (1.8) only. Do not current-inject this rail, see 3.1.

Not useful here, despite being the standard advice:

- Thermal camera. 28 mW produces nothing to see.
- Current injection and gradient tracing. Both need a sub-20 Ω short.

## Appendix: Capacitor Identification Notes

### 1. TOP SIDE IDENTIFIABLE PARTS

* **Broadcom BCM2712 SoC**: the large package marked `BROADCOM 2712...` is clearly visible.
* **LPDDR memory**: large BGA package immediately above the BCM2712. The package is visually identifiable as the memory device, but its exact marking is not sufficiently legible.
* **RP1 southbridge**: large Raspberry Pi-logo IC to the upper-right of the BCM2712.
* **PMIC**: Dialog device in the lower-left power section. `DA9091` is readable on the package.
* **Wireless module/shield**: shield can at upper-left. The shield/module reference designator is not legible.
* **Ethernet PHY area**: IC immediately to the lower-right of RP1, between RP1 and the Ethernet connector. Its detailed marking is **not legible**.
* **Ethernet magnetics/RJ45**: the large Traxcom Ethernet module is visible at lower-right. The actual magnetics are enclosed within the connector module, so individual transformer components cannot be inspected.
* **USB**: `J11 / USB2` and `J12 / USB3`.
* **GPIO**: `J8`.
* **PCIe FFC**: `J20 / PCIe`.
* **CAM/DISP**: `J3` and `J4`, with `CAM/DISP` silkscreen.
* **PoE**: `J14 / PoE`.
* **Ethernet**: `J10 / ETHERNET`.
* **USB-C power**: `J1`.
* **Battery connector**: `J5`, with `BAT` silkscreen.
* **HDMI**: `HDMI0` and `HDMI1`; `J2` and `J13` are visible around the two connectors.
* **UART**: `UART` silkscreen is visible. Its connector designator is not confidently legible.
* **Fan**: `FAN` silkscreen and the small connector are visible; the adjacent designator is difficult to read.
* Numerous small passives are visible around the PMIC, SoC, RP1, Ethernet PHY and I/O areas.

### 2. BOTTOM SIDE IDENTIFIABLE PARTS

* `J9 / SD CARD` is clearly visible.
* Very large numbers of test points are labelled `TPxx`. Examples clearly visible include `TP2`, `TP3`, `TP4`, `TP5`, `TP6`, `TP7`, `TP8`, `TP9`, `TP10`, `TP11`, `TP12`, `TP13`, `TP14`, `TP15`, `TP16`, `TP17`, `TP18`, `TP19`, `TP20`, `TP21`, `TP23`, `TP24`, `TP25`, `TP26`, `TP28`, `TP29`, `TP30`, `TP31`, `TP32`, `TP33`, `TP34`, `TP36`, `TP38`, `TP39`, `TP41`, `TP43`, `TP44`, `TP45`, `TP46`, `TP47`, `TP48`, `TP50`, and many of `TP53` through `TP76`.
* `FLASH WP` is clearly printed near the lower-central area.
* There is a small **8-pin IC** on the lower-right/central portion of the bottom side. Its marking is not legible. **I cannot prove from this photograph alone that this is the SPI flash.** That identification is only a low-confidence visual inference.
* The large central areas containing dense vias/fanout appear to correspond to areas beneath major IC packages, but I am **not assigning particular vias/passives to a particular net**.

### 3. Passive density and rework accessibility

| Region                               | Visible passive density | Hot-air accessibility                                  |
| ------------------------------------ | ----------------------- | ------------------------------------------------------ |
| **Top PMIC / power section**         | **High**                | Crowded, but components are exposed on the top surface |
| **Bottom-left dense passive field**  | **High**                | Crowded; many small parts/test points                  |
| **Bottom central BGA/routing field** | **Medium to high**      | Crowded, fine-pitch routing                            |
| **Top around BCM2712**               | **Medium**              | Adjacent to large package/BGA                          |
| **Top around RP1**                   | **Medium**              | Adjacent to BGA                                        |
| **Top around LPDDR**                 | **Low nearby**          | Immediately adjacent to BGA                            |
| **Ethernet PHY area**                | **Medium**              | Crowded, near connectors                               |
| **CAM/DISP / PoE area**              | **Medium**              | Crowded between connectors                             |
| **Bottom SD-card area**              | **Low to medium**       | Connector makes access restricted                      |
| **USB/HDMI connector areas**         | **Low to medium**       | Connector-adjacent / partially obstructed              |
| **GPIO perimeter**                   | **Low**                 | Relatively clear                                       |

### 4. Relative likelihood of finding a 3.3 V decoupling capacitor

This is **only a physical-placement inference**. The photographs cannot establish which capacitor is actually connected to 3.3 V.

1. **Top PMIC/power region** - highest visual priority. There is a conspicuous concentration of capacitors and inductive/power components around the `DA9091`.
2. **Bottom dense passive field** - very high priority. There are numerous small capacitors/resistors in a concentrated area, making this an important physical inspection zone.
3. **Around the BCM2712 / LPDDR / RP1** - high priority. Large digital ICs normally have substantial local decoupling, and the photograph visibly shows populated areas around these packages.
4. **Bottom central BGA/routing region** - high/medium priority. Dense passive population and extensive power/ground via structures are visible.
5. **Ethernet PHY region** - medium priority.
6. **CAM/DISP / PoE / I/O regions** - medium to low priority.
7. **USB, HDMI, SD-card and connector-only areas** - lower priority based strictly on visible passive density.

The important limitation is that **none of those observations identifies a capacitor as being on the 3.3 V rail**.

### 5. Anomalies

I do **not** see a feature that I would confidently classify from these photographs as:

* cracked/chipped MLCC
* obvious solder bridge
* obvious solder ball
* lifted component
* burned/discolored PCB
* gross tooling damage
* localized residue clearly distinguishable from normal flux/test-point residue

There are numerous bright/white-looking areas around test points and soldered features on the bottom side. I would **not call those residue or damage from these photographs** because they are also consistent with solder/test-point surfaces and photographic illumination.

### 5. Annotated photographs

I created full annotated inspection plates with numbered regions and a compact legend, including passive density, hot-air accessibility, relative 3.3 V inspection priority, and confidence.

[**Review annotated Raspberry Pi 5 top-side inspection**](img/pi5_top_annotated.png)

[**Review annotated Raspberry Pi 5 bottom-side inspection**](img/pi5_bottom_annotated.png)

### Most useful observation for your 393 Ω measurement

From the photographs alone, I would **not start removing random capacitors**. The visually concentrated areas worth examining first are the **PMIC/power section on the top** and the **large dense passive population on the bottom**. Those are the areas where the photographs give the strongest physical evidence for potentially relevant decoupling components.

But the photographs cannot tell us which of those capacitors is actually connected to your measured GPIO1/3.3 V rail. The `393 Ω` versus `24 kΩ` measurement is therefore useful as an electrical fault indicator, but it cannot be mapped to a particular capacitor from these images alone.
