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
| `BEND_RADIUS` | 6 | Rounds the ramp→channel corner so the cable isn't kinked; 0 = old sharp corner |
| `FACE_CHANNEL_LEN` | 12 | Open channel on the face, from the end of the bend inward |
| `FACE_DEPTH` | 0 | Face channel depth; 0 = `d` |
| `CONNECTOR_ALLOWANCE` | 30 | Cable+connector length per end stored in the user-drawn pocket |
| `EDGE_FILLET` | 1.5 | Top/bottom outer edge fillet; 0 = none |
| `SAMPLE_STEP` | 3.0 | Ramp sampling step; the spiral itself is exact geometry |
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
bend_len = BEND_RADIUS * pi/2               always a quarter turn (see below)
per_end  = hypot(RAMP_LENGTH, z0-zb) + bend_len + FACE_CHANNEL_LEN + CONNECTOR_ALLOWANCE
L_spiral = CABLE_LENGTH - 2*per_end
auto:   pitch = d + MIN_WALL; turns = L_spiral / hypot(P, pitch); H = 2*z0 + pitch*turns
fixed:  iterate turns/pitch with H given; error if pitch < d + MIN_WALL
```

Path: parameter `u` = distance along the stadium centre-line, `u = 0` at the
middle of the −Y straight side, counter-clockwise from the top (`stadium_xy`).
`u ∈ [−RAMP, 0)` is the bottom ramp, `[0, turns·P]` the spiral (linear z), and
`(turns·P, turns·P + RAMP]` the top ramp. Ramps use smoothstep in z, ending
horizontal at the face-channel height. The spine is built from exact geometry:
the straights are line segments, the round ends are true helix arcs (`Part.makeHelix`, rotated/translated onto each end), so the
spine is never a BSpline. Only the ramps are subdivided (every `SAMPLE_STEP`),
since z is not linear in u there, so each piece is one constant-pitch helix.
A circle of radius R is swept along the whole spine
(`makePipeShell`, corrected-Frenet mode). If that fails, the fallback sweeps
each spine edge individually and fuses them with spheres at the joints.

Corner bend (`arc_ending_at`): where the ramp reaches the face it has to turn
from the perimeter direction into the channel direction. `inward_dir` is
perpendicular to the stadium tangent everywhere (on the caps it points at the
cap centre, on the straights it is ±Y), so this corner is **always a quarter
turn** and its length is known before anything is built. A horizontal arc of
`BEND_RADIUS` is fitted tangent to both, appended to the spine wire, and swept
together with the rest of the groove - so the cable curves through instead of
being kinked around the groove radius. The bottom arc ends at the ramp start
(travelling *outward*, `-inward_dir`); the top arc is the same curve with both
tangents negated, which describes it traversed the other way.

Face channels (`face_channel`): start where the bend ends, and run inward along
the bend's exit direction (handed in, not re-derived, so the two meet without a
kink). Built from a horizontal cylinder, end spheres,
vertical cylinders and a rotated box → open U-slot. Anchors = channel end at
the face (z = 0 / H).

Body: stadium wire → face → extrude, fillet top/bottom edges, then cut the
groove, then each face channel as a single fused tool (see Known issues for why
each of those matters), then `removeSplitter` - but only if refining a *copy*
comes back valid, since it can quietly wreck the solid.

## Status

- Runs headlessly (`FreeCADCmd`) and in-app; verified across several parameter
  sets (default, cylinder body, fixed height, no fillet / half-open groove,
  thick cable, short cable) - each produces a single valid solid.
- Targets FreeCAD 0.20 / 0.21 / 1.0 (PySide6 → PySide2 → PySide fallback).

## Known issues / TODO

- [x] ~~Face channel meets the ramp with a sharp 90° turn in plan view.~~
      Fixed: `BEND_RADIUS` (default 6 mm) fits a tangent quarter-turn arc into
      the spine between ramp and channel. Validated so the channel can't run
      out through the far wall (it silently ate the whole part before).
- [x] ~~Test in FreeCAD; verify sweep validity and boolean speed for long cables.~~
      Fixed: a BSpline-interpolated spine made `makePipeShell` raise
      `BRepOffsetAPI_MakePipeShell::MakeSolid` on this OCC build (reproduced
      headlessly on FreeCAD 26.3.0 / 1.1dev weekly). The centre-line is now
      built from exact line/helix edges instead of a sampled+interpolated
      BSpline, which sweeps reliably. Also: cutting the groove fused with the
      face channels in one boolean op could yield an invalid/multi-solid
      result (tool self-touches near the ramps); the macro now cuts the
      groove first, then each face channel as its own fused tool.
- [ ] Optional: generate a default USB-C pocket (parameters: plug width/thickness/length)
      instead of leaving it fully manual.
- [ ] Optional: drive parameters from a FreeCAD Spreadsheet so the model is
      recomputable without re-running the macro (or convert to a FeaturePython object).
- [ ] Re-running creates duplicate objects – add cleanup/update of existing objects.
- [ ] Cancelling the dialog raises `Cancelled` (shows as an error in the Report view) –
      handle it gracefully.
- [ ] Validate that `FACE_CHANNEL_LEN` plus the pocket fit on the face (warns only).
- [ ] Meshing the result (`MeshPart`/`tessellate`) reports a non-closed,
      self-intersecting mesh even though the BRep is a valid closed single
      solid. Pre-existing (same with `BEND_RADIUS = 0`); check before relying on
      a direct STL export for printing.
- [ ] Headless test script (`FreeCADCmd`) that builds several parameter sets and
      checks `shape.isValid()`, volume and bounding box.

## Printing notes

With `GROOVE_DEPTH_FAC` > 0.5 the groove is undercut, so its opening at the
surface is narrower than Ø`d` and the rib you see between turns is wider than
`MIN_WALL` (`MIN_WALL` is the true minimum, and it sits below the surface). The
macro prints all three numbers each run; lower `GROOVE_DEPTH_FAC` for a wider
opening and tighter looking coils.

Print standing on one of the flat faces. The groove is then a sequence of
horizontal-ish overhangs; with the snap-in lip (`GROOVE_DEPTH_FAC` > 0.5)
check the overhang, or print on its side for a cleaner groove. Tune `CLEARANCE`
after a test print.
