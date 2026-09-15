# CableSpool – parametric spiral cable holder (FreeCAD macro)

A FreeCAD macro that generates a stadium-shaped (rectangle with semicircle ends)
cable spool. A helical snap-in groove wraps the side wall; the first and last
loops continue into ramp "corridors" that reach the bottom and top faces, where
the user draws the connector pockets (e.g. USB-C) by hand.

Inspired by 3D-printed cable organizers where the cable is wound around the
body and both connectors are stored flush in the top/bottom faces.

## Files

| File | Purpose |
|---|---|
| `CableSpool.FCMacro` | The macro (Python, runs inside FreeCAD) |
| `README.md` | This document |

## Usage

1. Copy `CableSpool.FCMacro` to your FreeCAD macro folder
   (Macro → Macros… shows the path) or open it directly.
2. Run it. A dialog asks for the parameters (values are remembered between runs).
3. The macro creates:
   - `CableSpool` – PartDesign Body whose BaseFeature is the generated shape
   - `CableSpool_Base` – the generated Part shape (inside the Body)
   - `SpiralPath` – the groove centre-line (hidden), useful as reference
   - `Anchor_Bottom`, `Anchor_Top` – red points at the inner end of each face channel
4. Sketch on the top/bottom face, use External Geometry to reference the anchor,
   draw the connector outline starting there and Pocket it.
5. The Report view prints turns, pitch, height, cable-length breakdown and
   anchor coordinates.

Set `USE_DIALOG = False` to use only the constants at the top of the file
(also the behaviour when FreeCAD runs without a GUI).

## Parameters (mm)

| Name | Default | Meaning |
|---|---|---|
| `CABLE_LENGTH` | 1500 | Total cable length incl. connectors |
| `CABLE_DIAMETER` | 3.5 | Measured cable diameter |
| `CLEARANCE` | 0.3 | Added to diameter → groove diameter `d` |
| `GROOVE_DEPTH_FAC` | 0.85 | Groove depth / `d`. 0.5 = half open, >0.5 = snap-in lip, must be < 1 |
| `MIN_WALL` | 1.2 | Rib between neighbouring turns |
| `WIDTH` | 40 | Outer width = diameter of the round ends |
| `LENGTH` | 75 | Outer overall length (≥ WIDTH; equal → cylinder) |
| `HEIGHT` | 0 | 0 = auto from cable; >0 = fixed height, pitch adapts |
| `END_MARGIN` | 0 | Face → centre of first/last turn; 0 = auto |
| `RAMP_LENGTH` | 25 | Perimeter length used by the corridor to climb to a face |
| `FACE_CHANNEL_LEN` | 12 | Open channel on the face, from edge inward |
| `FACE_DEPTH` | 0 | Face channel depth; 0 = `d` |
| `CONNECTOR_ALLOWANCE` | 30 | Cable+connector length per end stored in the user-drawn pocket |
| `EDGE_FILLET` | 1.5 | Top/bottom outer edge fillet; 0 = none |
| `SAMPLE_STEP` | 3.0 | Path sampling step for the spline |
| `USE_DIALOG` | True | Show Qt input dialog |

## Algorithm

Derived values:

```
d      = CABLE_DIAMETER + CLEARANCE          groove diameter, R = d/2
depth  = GROOVE_DEPTH_FAC * d
r      = WIDTH/2 - depth + R                 groove centre-line radius
S      = (LENGTH - WIDTH)/2                  half straight length
P      = 4*S + 2*pi*r                        centre-line perimeter per turn
zb     = face_depth - R                      centre z of bottom face channel
z0     = END_MARGIN or face_depth + MIN_WALL + R
per_end  = hypot(RAMP_LENGTH, z0-zb) + FACE_CHANNEL_LEN + CONNECTOR_ALLOWANCE
L_spiral = CABLE_LENGTH - 2*per_end
auto:   pitch = d + MIN_WALL; turns = L_spiral / hypot(P, pitch); H = 2*z0 + pitch*turns
fixed:  iterate turns/pitch with H given; error if pitch < d + MIN_WALL
```

Path: parameter `u` = distance along the stadium centre-line, `u = 0` at the
middle of the −Y straight side, counter-clockwise from the top (`stadium_xy`).
`u ∈ [−RAMP, 0)` is the bottom ramp, `[0, turns·P]` the spiral (linear z), and
`(turns·P, turns·P + RAMP]` the top ramp. Ramps use smoothstep in z, ending
horizontal at the face-channel height. Points are sampled every `SAMPLE_STEP`,
built from exact geometry: the straights are line segments, the round ends are
true helix arcs (`Part.makeHelix`, rotated/translated onto each end), so the
spine is never a BSpline. A circle of radius R is swept along it
(`makePipeShell`, corrected-Frenet mode). If that fails, the fallback sweeps
each spine edge individually and fuses them with spheres at the joints.

Face channels (`face_channel`): from the ramp end, go inward toward the nearest
point of the body's centre segment. Built from a horizontal cylinder, end spheres,
vertical cylinders and a rotated box → open U-slot. Anchors = channel end at
the face (z = 0 / H).

Body: stadium wire → face → extrude, fillet top/bottom edges, then cut the
groove and each face channel one at a time (not fused/multi-cut first - see
Known issues), then `removeSplitter`.

## Status

- Runs headlessly (`FreeCADCmd`) and in-app; verified across several parameter
  sets (default, cylinder body, fixed height, no fillet / half-open groove,
  thick cable, short cable) - each produces a single valid solid.
- Targets FreeCAD 0.20 / 0.21 / 1.0 (PySide6 → PySide2 → PySide fallback).

## Known issues / TODO

- [x] ~~Test in FreeCAD; verify sweep validity and boolean speed for long cables.~~
      Fixed: a BSpline-interpolated spine made `makePipeShell` raise
      `BRepOffsetAPI_MakePipeShell::MakeSolid` on this OCC build (reproduced
      headlessly on FreeCAD 26.3.0 / 1.1dev weekly). The centre-line is now
      built from exact line/helix edges instead of a sampled+interpolated
      BSpline, which sweeps reliably. Also: cutting the groove fused with the
      face channels in one boolean op could yield an invalid/multi-solid
      result (tool self-touches near the ramps); the macro now cuts the
      groove and each channel one at a time.
- [ ] Face channel meets the ramp with a sharp 90° turn in plan view – consider an
      arc (bend radius parameter) for stiff cables.
- [ ] Optional: generate a default USB-C pocket (parameters: plug width/thickness/length)
      instead of leaving it fully manual.
- [ ] Optional: drive parameters from a FreeCAD Spreadsheet so the model is
      recomputable without re-running the macro (or convert to a FeaturePython object).
- [ ] Re-running creates duplicate objects – add cleanup/update of existing objects.
- [ ] Cancelling the dialog raises `Cancelled` (shows as an error in the Report view) –
      handle it gracefully.
- [ ] Validate that `FACE_CHANNEL_LEN` plus the pocket fit on the face (warns only).
- [ ] Headless test script (`FreeCADCmd`) that builds several parameter sets and
      checks `shape.isValid()`, volume and bounding box.

## Printing notes

Print standing on one of the flat faces. The groove is then a sequence of
horizontal-ish overhangs; with the snap-in lip (`GROOVE_DEPTH_FAC` > 0.5)
check the overhang, or print on its side for a cleaner groove. Tune `CLEARANCE`
after a test print.
