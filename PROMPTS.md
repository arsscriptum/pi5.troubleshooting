# Image and Diagram Prompts

Prompts for generating the figures that go with [Section 2](README.md#2-finding-the-shorted-capacitor-on-the-33v-rail).

## 0. Read This First

No image generator has an accurate internal model of the Raspberry Pi 5's component placement. Anything it draws from a text prompt alone is a plausible-looking fiction: capacitor positions, counts, designators and package sizes will all be invented, and they will look convincing.

- **Never rework from a generated image.** The only trustworthy cap map is the one you build yourself with a meter against the good board, per 2.3.
- Generated images are for documentation, orientation and figures. Nothing else.
- Prompt A below is the exception and the one to prefer, because it works from your own photographs rather than from the model's imagination.

---

## A. Annotate Your Own Board Photos (most accurate — use this one)

For ChatGPT with image upload. Attach `img/20261001_112322.jpg` (top side) and `img/20261001_112335.jpg` (bottom side).

```
I am troubleshooting a Raspberry Pi 5 with a partial short on the 3.3V rail
(393 ohms from GPIO pin 1 to pin 6, versus 24 kOhm on a known-good board).

I have attached two photographs of the board: one of the top side, one of the
bottom side.

Working ONLY from what is visible in these photographs, do the following:

1. Identify and label every major component you can actually see and read:
   the SoC, the LPDDR memory package, the RP1 southbridge, the PMIC, the
   wireless module shield, the Ethernet magnetics and PHY, the SPI flash,
   connectors (USB-C, HDMI, CAM/DISP, PCIe FFC, GPIO header, SD slot, fan,
   RTC battery, PoE, UART), and the silkscreen reference designators and test
   point labels that are legible.

2. Divide each side into labelled regions by function, and for each region
   state the approximate density of small passive components you can see,
   and how physically accessible those passives are for hot-air rework
   (clear area / crowded / under a connector / adjacent to a BGA).

3. Rank the regions by how likely they are to host decoupling capacitors on
   a 3.3V rail, and explain your reasoning from what is visible.

4. Flag anything that looks anomalous: residue, discolouration, cracked or
   chipped components, solder balls, bridges, lifted parts, tooling damage.
   Give me the region and the nearest legible silkscreen marking for each.

5. Produce an annotated version of each image with the regions outlined and
   labelled, and a numbered legend.

Rules:
- Do not guess component designators or capacitor positions that are not
  legible in the photos. If you cannot read it, say "not legible".
- Do not invent a schematic or a net list. There is no published full
  schematic for the Pi 5 and I know it.
- Where you are inferring rather than reading, mark it clearly as an
  inference and give your confidence.
```

Follow-up prompt once it has annotated the photos:

```
Now produce a blank worksheet I can print: a table with columns
# | Side | Region | Nearest silkscreen marking | Package size | On 3V3? (<1 ohm to pin 1) | Other pad to GND? | Notes
pre-filled with one row per region you identified, so I can fill in the
measurement results as I sweep the known-good board with a meter.
```

---

## B. 3.3V Rail Topology Diagram (conceptual — safe to generate)

This one is safe because it is a block diagram of how the rail is organised, not a claim about physical placement. Ask for SVG rather than a raster image so the text stays sharp and you can edit it.

```
Draw me a clean technical block diagram as SVG, suitable for a troubleshooting
document, showing the 3.3V rail topology of a Raspberry Pi 5.

Structure:
- USB-C 5V input on the left, into the PMIC (Renesas DA9091).
- From the PMIC, show the 3.3V buck output as a single thick net that fans
  out to the right.
- Show the other PMIC rails (0.8V core, 1.1V DDR, 1.8V) as separate stubs
  going off-diagram, clearly visually distinct from the 3.3V net, labelled
  "other rails - do NOT pull caps from these".
- On the 3.3V net, show these loads as blocks, each with a small decoupling
  capacitor symbol to ground beneath it: GPIO header 3V3 pins, SPI boot
  flash, SD card interface, RP1 southbridge I/O, Ethernet PHY, camera and
  display connectors, PCIe FFC connector, HDMI, RTC.
- Mark the two measurement access points used in the procedure: GPIO pin 1
  (3V3) and GPIO pin 6 (GND), with a DMM symbol across them.
- Annotate the fault: a resistor symbol of unknown location bridging the
  3.3V net to ground, labelled "fault: 393 ohms, location unknown
  (good board: 24 kOhm)".

Style: flat, monochrome plus one accent colour for the 3.3V net, thin lines,
sans-serif labels, no 3D, no shadows, no decorative elements, high contrast,
readable when printed in black and white.

Caption it: "Conceptual topology only - not a board layout. Load list is
indicative; verify every net by continuity against a known-good board."
```

If you want it as text instead of SVG, replace the first line with: `Give me this as a Mermaid flowchart.`

---

## C. Illustrative Region Figure (image generation — decorative only)

Use only as a chapter opener or an orientation figure. It will not be accurate.

```
A clean, flat, technical illustration of a single-board computer's printed
circuit board, viewed straight down, in the style of a service manual figure.
Green PCB, silver and black component packages, white silkscreen.

Overlay five translucent coloured region boxes with numbered callout labels
pointing to them, in the style of an exploded service diagram:
1. Power management cluster
2. Processor decoupling field
3. I/O controller
4. Flash and storage interface
5. Connector rails

Flat vector style, no photorealism, no text other than the numbered callouts,
no brand names or logos, high contrast, white background, suitable as a
figure in a printed troubleshooting document.
```

Note the deliberate omissions: no Raspberry Pi branding, no specific component
counts, no designators. Asking for those is what produces confident-looking
false detail.

---

## D. Capacitor Failure Mode Reference Figure

```
Draw a clean technical figure as SVG showing the three failure modes of a
surface-mount multilayer ceramic capacitor, side by side, each as a labelled
cross-section of an MLCC on a PCB pad:

1. "Flex crack" - a diagonal crack propagating from the termination through
   the dielectric layers, caused by board bending.
2. "Thermal crack" - a crack from thermal cycling stress, with the leakage
   path through the crack marked as a dashed line.
3. "Healthy" - intact dielectric layers for comparison.

Under each, show the expected DMM reading across the part:
hard short (0-20 ohms), leaky (200 ohms - 2 kOhm), open (megohms).

Style: flat, monochrome with one accent colour for the crack and leakage
path, thin lines, sans-serif labels, no shadows, printable in black and white.
```
