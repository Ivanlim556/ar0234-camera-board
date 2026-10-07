# AR0234 camera board (in-house)

A 35 × 35 mm global-shutter **mono** camera board for the drone, built around the onsemi **AR0234** (1920×1200,
MIPI CSI-2, 4 lanes). It plugs into the Raspberry Pi 5's CAM/DISP 0 with a 22-pin ribbon and is working since
**2026-10-05** (1280×800 at 30 fps, ~180 ms glass to glass — see the main README and `docs/guide.html` 2.0).

Designed by Lim Wei Quan (Gamuda internship, 2026). Board revision **2026-09-13**.

| Front (sensor) | Back (J1 ribbon connector) |
|---|---|
| ![front](photos/board-front-sensor.jpeg) | ![back](photos/board-back-j1.jpeg) |

| With M12 holder + 70° lens, on the Pi 5 + AI HAT+ | Bench setup with the VL53L5CX distance sensor |
|---|---|
| ![lens](photos/board-with-lens-on-pi5.jpeg) | ![bench](photos/bench-ar0234-with-vl53l5cx.jpeg) |

## Design (before fabrication)

EasyEDA Pro 3D view of the final design:

| Top — sensor side (AR0234, lens-holder holes) | Bottom — components |
|---|---|
| ![3D top](design/3d-top-sensor-side.png) | ![3D bottom](design/3d-bottom-components.png) |

Rendered from the exact Gerber files sent to JLCPCB (`fabrication/Gerber_ar0234-camera_2026-09-13.zip`, drawn with
[gerbonara](https://gitlab.com/gerbolyze/gerbonara); solder mask coloured blue to match the boards that came back).

| Top — sensor side (U1 AR0234, lens-holder holes) | Bottom — components (J1 ribbon, J2, regulators, clock, level shifter) |
|---|---|
| ![top render](design/render-top-sensor-side.png) | ![bottom render](design/render-bottom-components.png) |

Designed in EasyEDA Pro (schematic → 4-layer PCB, DRC, one-click Gerber/BOM/CPL export). The bottom render is seen
from below, as in the photo of the back.

## Files

| File | What |
|---|---|
| `SCH_ar0234-camera_2026-09-13.pdf` | Schematic |
| `fabrication/Gerber_ar0234-camera_2026-09-13.zip` | Gerber + drill files as sent for manufacture (4 layers) |
| `fabrication/BOM_ar0234-camera_PCB2_2026-09-13.xlsx` | Bill of materials, 23 lines / 55 parts, with LCSC part numbers |
| `fabrication/PickAndPlace_PCB2_2026_09_13.xlsx` | Component placement (CPL) for assembly |
| `fabrication/Netlist_ar0234-camera_2026-09-13.tel` | Netlist (handy for finding test points) |
| `../../pi/ar0234/ar0234-gamuda-power-overlay.dts` | Pi 5 device-tree overlay for this board's power-up timing |

## How it was made

- **Design:** EasyEDA Pro v3.2.148 (schematic + PCB). Gerbers exported 2026-09-13.
- **Fabrication + assembly:** JLCPCB — PCB and SMT assembly from the Gerber, BOM and CPL above; all parts from LCSC
  (JLCPCB's parts house), so every BOM line carries an LCSC number (`Cxxxxx`).
- **Board:** 4 layers, 1.585 mm, 35 × 35 mm, mounting holes 20 mm apart, 55 placed parts, 0.20 mm drills ×134.

## Circuit in short

| Block | Parts |
|---|---|
| Sensor | U1 **AR0234CSSM00SUKA0-CP** (mono, ODCSP-83), I²C address **0x10** |
| Power in | **+3V3** from the Pi on J1 pin 22 |
| Rails | U2 LD39050 → **+2V8** (115 mA) · U3 LD39050 → **+1V8** (13 mA) · U4 TLV62568 buck + L1 2.2 µH → **+1V2** (250 mA) |
| Power sequencing | EN_RAW (J1.17) → D1 BAT54C + RC delays: 2.8 V, then 1.8 V, then 1.2 V; reset released via R11/C14 |
| Clock | Y1 24.000 MHz oscillator → EXTCLK via R17 33 Ω |
| I²C | U5 TCA39306 level shifter (3.3 V ribbon side ↔ 1.8 V sensor side), R7/R8 2.7 kΩ pull-ups |
| Connectors | J1 Hirose FH12-22S-0.5SH (22-pin, 0.5 mm, Pi 5 pinout, 4 lanes) · J2 3-pin: 1 FLASH, 2 TRIGGER, 3 GND |

**Test points** (big 10 µF caps, rail to GND): **C24** = +3V3 in · **C21** = +2V8 · **C22** = +1V8 · **C23** = +1V2 ·
GND at **J2 pin 3**.

## Bring-up notes (arrival day, 2026-10-05)

1. **Cable direction matters.** The ribbon's bare contacts must face the pads at **both** ends (blue stiffener away
   from the board at J1). The wrong way gives 0 V on C24 and `failed to read chip id`.
2. **The mono sensor reports the colour chip ID `0x0A56`** (the driver expects `0x1A56` for mono). Raw frames prove
   it is mono. Fix: `pi/ar0234/ar0234-force-mono.patch` + `options ar0234 force_mono=1`.
3. Driver: Kurokesu `ar0234-rpi-dkms` + their libcamera (standard libcamera has no AR0234). Full steps: main README,
   "AR0234 setup (v2.0)".

## Known limits of this revision

- **TRIGGER (J2 pin 2) has no pull-down** — it floats. Master mode (the default) ignores it; don't use
  external-trigger / sync-sink modes without adding 10 kΩ from J2 pin 2 to pin 3.
- **MIPI traces are 0.120 mm** where 100 Ω wants 0.127 mm; pairs matched under 0.10 mm. Fine at the tested
  450 MHz link; suspect it first if a long cable or higher rate gives CRC errors.
- **R7/R8 pull-ups** sit in parallel with the Pi's (~1.2 kΩ combined) — legal at 400 kHz.
- **FLASH (J1 pin 18)** is a sensor output on a pin the Pi may also drive: tape contact 18 at the camera end, or use
  FLASH from J2 pin 1.

## Further improvements (next revision)

Lessons from bringing up two boards (October 2026), in order of impact.

### 1. Centre the pixel array — not the package — under the lens

The AR0234's light-sensitive area is **not in the middle of its package**. onsemi's package drawing (datasheet
p. 24, note 12): *"Optical center relative to package center (X, Y) = (−1.271, −0.148) mm"* — the 1928 × 1208
active array sits toward the **ball-A1 end** of the 9.995 mm package.

| This revision (rev 2026-09-13) | X (mm) | Y (mm) |
|---|---|---|
| Lens-holder holes — midpoint = lens axis | 17.500 | 18.135 |
| U1 package centre (pick-and-place) | 18.542 | 18.161 |
| Package centre − lens axis | +1.042 | +0.026 |

Depending on which end of U1 the A1 corner is, the pixel-array centre is either **≈0.23 mm** from the lens axis (the
1.04 mm shift mostly cancels the 1.27 mm) or **≈2.3 mm** (they add). With the 4 mm lens, 1 mm of offset turns the
view by ~14°: 0.23 mm ≈ 3° (≈50 px of the 1280-wide picture), 2.3 mm ≈ 30°. Camera-vs-distance-sensor tests on
2026-10-05/07 agreed within 1–2 zones (mostly parallax), which fits the **small (~3°)** case — but a person
"straight in front" still shows slightly off-centre.

**Fix:** place U1 so the **optical centre** lands on the holder-hole midpoint: package centre = lens axis +
(1.271, 0.148) mm rotated into the board frame, on the side **away from A1**. Check the A1 marker in the footprint
against the datasheet top view before moving it, and confirm with a measurement (camera square to a wall, mark the
point in line with the lens, see where it lands in the picture).

### 2. Mark the ribbon direction on the board

The first power-up failed (0 V on the board, `failed to read chip id`) only because the 22-pin ribbon was the wrong
way round. Add silkscreen next to J1: **"contacts ↓ / blue stiffener up"** and a pin-1 arrow.

### 3. Labelled test pads

Bring-up used the 10 µF caps as probe points (C24 = 3V3, C21 = 2V8, C22 = 1V8, C23 = 1V2). Add labelled test pads
for **3V3, 2V8, 1V8, 1V2, EXTCLK, GND** in one row along an edge.

### 4. Fix the known signal issues

- **TRIGGER:** 10 kΩ pull-down on J2 pin 2 (it floats today).
- **MIPI pairs:** 0.127 mm traces for 100 Ω differential on this stack-up (0.120 mm today).
- **FLASH:** don't route the sensor's FLASH output to J1 pin 18 (the Pi may drive CAM_IO1) — keep it on J2 only, or
  add a 0 Ω/DNP link so contact 18 is open by default (no Kapton tape needed).

### 5. Put the distance sensor on the camera board

The VL53L5CX lives on a separate SATEL board with 9 jumper wires: they came loose three times in a week, and its
position relative to the camera (beside it, turned 90°) needed a hand-tuned grid orientation and causes parallax at
close range. A VL53L5CX footprint **right next to the lens**, same orientation as the sensor rows, sharing the
board's 3V3/I²C (separate address 0x29) and one connector, would fix all three: no jumpers, fixed alignment,
minimal parallax.

### 6. Mechanics for the drone

- Mounting holes for a rigid camera mount and **strain relief for the ribbon** (a loose ribbon gave
  `Camera frontend has timed out` on 2026-10-06).
- A lens-holder **lock** (thread-lock pad or set-screw holder) so vibration can't turn the focus.
- Keep the same holder type across boards: a pre-focused holder + lens moved from board 1 to board 2 stayed sharp.

### 7. Software notes that come from the hardware

- The fitted mono part reports the **colour** chip ID `0x0A56`; until the driver learns the difference, the
  `force_mono` patch stays (see `pi/ar0234/`).
- The 35 ms power-up delay lives in a device-tree overlay. A supervisor that holds RESET_BAR low until +1V2 is good
  would make the board independent of that overlay.
