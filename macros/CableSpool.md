# CableSpool – parametric spiral cable holder (FreeCAD macro)

A FreeCAD macro that generates a stadium-shaped (rectangle with semicircle ends)
cable spool. A helical snap-in groove wraps the side wall; the first and last
loops continue into ramp "corridors" that reach the bottom and top faces. There
the groove keeps running round while tightening inward - the way the last turn
of a wound cable does - so it ends up on the inside of the wall, where the user
draws the connector pockets (e.g. USB-C) by hand.

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
   Clicking a field draws what that parameter means, to scale, from the values
   currently in the dialog - a wall cross-section (cable, clearance, groove
   depth, wall/pitch), a top view (outline, ramp, corner bend, face channel,
   pocket) or a side view (height, end margin, turns). A line under the drawing
   shows the resulting groove Ø, pitch, turn count and height before you build.
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
(also the behaviour when FreeCAD runs without a GUI). The drawings are painted
with QPainter from the live values (`preview_geometry` repeats the macro's own
derivation), so there are no image files to ship alongside the macro.

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
| `FACE_TURN` | 0.1 | *Minimum* laps of turn-in; it is stretched past this to wherever the straight that follows comes out longest. 0 = stop at the wall |
| `FACE_INSET` | 0 | How far inward the turn-in moves; 0 = auto, which aims it at the middle of the face |
| `FACE_EXIT_ANGLE` | 35 | Angle off the long axis the face channel leaves at |
| `FACE_BEND` | 12 | Radius of the arc that swings it onto that heading |
| `SLOT_EPS` | 0.05 | Face channel oversize, keeps its walls off the swept tube |
| `FACE_DEPTH` | 0 | Face channel depth; 0 = `d` |
| `CONNECTOR_ALLOWANCE` | 20 | Clear run kept at the end of the channel for the connector. Lower it and the channel runs further |
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
face_inset = FACE_INSET or r - max(2*d, R + MIN_WALL + 0.5)
face_span  = >= FACE_TURN*P, stretched to wherever the tail comes out longest
tail_len   = clear run across the face, less CONNECTOR_ALLOWANCE
face_len   = turn-in length + release arc + tail_len
per_end  = hypot(RAMP_LENGTH, z0-zb) + face_len + CONNECTOR_ALLOWANCE
turns    = snapped down to a half turn (makes both ends congruent)
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

Face runs (`off_of`, `face_slot`): past the ramp the groove does not strike out
for the centre - it keeps going round the stadium while easing inward by
`face_inset` (same smoothstep the ramps use, so the join is tangent-continuous
and there is no corner anywhere). Offsetting a stadium point toward the centre
segment keeps the same inward direction, so the offset path is just the same
stadium at a shrinking radius.

The swept tube **stops** where the ramp reaches the face; the face run itself is
a prismatic channel - a ribbon of width `d` around the path, extruded through
the face. Keeping the two apart is deliberate: when the tube ran alongside the
slot for the whole run their surfaces were tangent, and the kernel answered that
with a *larger* volume than it started with and a sealed void inside the part.
The channel is `SLOT_EPS` oversize and overlaps the end of the tube, so there
the tube sits strictly inside it - a clean crossing instead of a tangency - and
it is built in overlapping chunks and fused, because as a single polygon the
ribbon self-intersects once the run laps back over itself. The release arc and
straight are built separately from the turn-in for the same reason: they cut
back across the middle, so sharing a chunk can put a crossing inside one
polygon. They are also resampled to even spacing first - the arc samples sit
~1 mm apart and the straight arrives as one long jump, and the ribbon normal at
that join averages two very different directions and twists the quad. The
turn-in is oriented so the end the arc hangs off comes first, because the two
ends are walked in opposite directions and otherwise the tail is stitched to the
ramp end and the ribbon jumps clean across the part.

After the turn-in the run is swung by a `FACE_BEND` arc onto a diagonal
(`FACE_EXIT_ANGLE` off the long axis) and then runs straight across the open
face (`face_plan`). That straight is what gives the connector room - wrapping on
round the outline instead just parcels the pocket up in more channel.

The release arc is not optional: a tangent to the turn-in points *along* the
perimeter, so simply carrying on straight meets the wall within a dozen mm. And
running down the long axis instead, which reaches furthest, leaves the straight
lying in a lane with the channel beside it - so the clear run is measured with a
groove's spacing of clearance from the rest of the run, not just to the outer
wall.

It stops `CONNECTOR_ALLOWANCE` short of the far side, so the pocket you sketch
from the anchor has exactly that much in front of it. That parameter is the one
lever on how far the channel runs: every mm off it is a mm more channel, and the
anchor moves with it. Where the turn-in stops decides where the diagonal starts,
so every phase of it over one lap is tried and the roomiest kept.

`turns` is then **snapped down to a half turn**. `stadium_xy(-u)` is
`stadium_xy(u)` mirrored in X, and `u → u + P/2` is the stadium's own 180°
symmetry, so at a whole or half turn the top run is congruent to the bottom one:
same length, same room, ending symmetrically. That is also what makes this
solvable in one pass - the top run depends on where the spiral stops, which
depends on the budget, which depends on the run, and since the best phase jumps
around, iterating never settled and quietly left the two ends inconsistent.
Rounding down spends slightly less cable than `CABLE_LENGTH`; the Report view
prints what was actually used and how much is spare.

`face_plan` is a pure function of the numbers and the dialog preview calls the
same one, so the preview's turn count and height cannot drift from the build's.

Anchors = the far end of each face run (turn-in + straight), on the face.

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
      Fixed twice: first by rounding the corner with a `BEND_RADIUS` arc, then
      properly by dropping the corner altogether - the groove now continues
      round the face and tightens inward, as on the commercial winders this is
      modelled on. `FACE_TURN`/`FACE_INSET` replace `BEND_RADIUS` and
      `FACE_CHANNEL_LEN`. Guarded so a run that laps back over itself, or one
      inset deeper than the spool allows, fails with a message instead of
      quietly eating the part.
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
- [ ] Validate that the face run plus the pocket you draw actually fit on the
      face (the run itself is checked, the pocket is not).
- [ ] The dialog diagrams cover every parameter except `FACE_DEPTH`,
      `EDGE_FILLET`, `END_MARGIN` and `SAMPLE_STEP`, which are file constants
      rather than dialog fields.
- [ ] Meshing the result (`MeshPart`/`tessellate`) reports a non-closed,
      self-intersecting mesh even though the BRep is a valid closed single
      solid. Pre-existing (same with `FACE_TURN = 0`); check before relying on
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
