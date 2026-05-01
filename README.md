# FreeCAD Tools

A collection of FreeCAD macros for mechanical design automation.

---

## Macros

### TrackChain — Track Link Chain Generator

Generates a complete track-link chain (tank / conveyor style) as a FreeCAD Part compound.  
The chain is laid out as an oval with two straight runs and two semicircular ends.

---

### Chain layout

![Chain Layout](docs/chain-layout.svg)

> **R** = arc radius · **pitch** = joint-to-joint distance · arrows show CCW direction

---

### Features

| | |
|---|---|
| Interactive joint picker | Click hole edges in the 3-D view — no manual coordinate entry |
| Live pitch readout | Displays joint-to-joint distance as soon as both joints are picked |
| Loop direction | Up (CCW) or Down (CW) — switch when the chain wraps the wrong way |
| Oval or circular | Set straight-section count to `0` for a pure circle |
| Exact joint alignment | Chord-based arc radius ensures every pin hole meets its neighbour precisely |

---

### Requirements

| Dependency | Version |
|---|---|
| FreeCAD | 0.20 or later |
| PySide2 | bundled with FreeCAD |

---

### Installation

Copy `macros/TrackChain.FCMacro` into your FreeCAD macros folder:

| Platform | Path |
|---|---|
| Linux | `~/.local/share/FreeCAD/Macro/` |
| macOS | `~/Library/Application Support/FreeCAD/Macro/` |
| Windows | `%APPDATA%\FreeCAD\Macro\` |

Then open it via **Macro → Macros…** in FreeCAD.

---

### Usage workflow

```mermaid
flowchart LR
    A([Select body\nin 3-D view]) --> B([Run macro])
    B --> C([Click LEFT hole edge\nthen ⬅ Pick Left Joint])
    C --> D([Click RIGHT hole edge\nthen Pick Right Joint ➡])
    D --> E([Choose direction\n↑ Up · ↓ Down])
    E --> F([Set total links N\nand N_straight])
    F --> G([Generate Chain])
    G --> H([TrackChain object\nadded to document])
```

The dialog is **non-modal** — the 3-D view stays fully interactive while it is open.

---

### Parameters

| Parameter | Range | Notes |
|---|---|---|
| Total links | 4 – 9998 (even) | Odd values are rounded down automatically |
| Links per straight side | 0 – N/2−2 | `0` = pure circle, higher = longer oval |
| Loop direction | Up / Down | **Up (CCW)** — curves above the straights; **Down (CW)** — curves below |

---

### Geometry

The oval consists of four sections:

```
         ← N_straight links →
    ●────────────────────────●   ← top straight
   ╱                          ╲
  ╱   N_per_curve links each   ╲  ← arcs
  ╲                            ╱
   ╲                          ╱
    ●────────────────────────●   ← bottom straight
```

| Symbol | Formula | Description |
|---|---|---|
| `pitch` | `\|j2 − j1\|` | Distance between the two picked joint centres |
| `R` | `pitch / (2 · sin(π / (2 · N_per_curve)))` | Arc radius — circumradius of the inscribed polygon with side = pitch |
| `straight length` | `N_straight · pitch` | Length of each straight run |

Using the chord-based radius ensures the pin-to-pin chord on the arc equals exactly one `pitch`, so every joint hole coincides perfectly with its neighbour.

The direction multiplier `D = ±1` mirrors the entire layout in Y — no geometry is recomputed, the whole chain flips.

---

### Output

A `Part::Feature` named `TrackChain_<N>links_<up|down>` is added to the active document as a compound of `N` transformed copies of the source link's `Shape`.

---

## License

MIT
