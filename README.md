# UR20 Teach Pendant Simulator

> A browser-based UR20 robot teach pendant — program, preview, and export PolyScope X programs with zero installation.

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)
![Three.js](https://img.shields.io/badge/Three.js-r158-blue.svg)
![Dependencies](https://img.shields.io/badge/dependencies-none-brightgreen.svg)
![Single File](https://img.shields.io/badge/build-single%20file-orange.svg)

---

A fully client-side UR20 robot simulator that runs in any modern browser. No server, no install, no build step. Open one HTML file and you have a functional 6-DOF robot teach pendant with forward/inverse kinematics, 3D visualization, waypoint programming, and export to Universal Robots' native PolyScope X format.

> **Screenshot:** *(place a screenshot of the simulator here — `docs/screenshot.png`)*

---

## Features

### Kinematics
- Full 6-DOF forward kinematics via standard DH convention (4×4 matrix chain)
- Numerical inverse kinematics using Levenberg-Marquardt with 200-iteration convergence, 0.1 mm tolerance
- Live IK on Cartesian XYZ slider drag — joint angles update in real time
- Singularity-safe: invalid IK targets are silently ignored, last valid state held

### 3D Visualization
- Hardware-accelerated WebGL via Three.js r158
- Schematic robot model: sphere joints + cylinder links, updates every frame
- PCFSoft shadow mapping, ACES filmic tone mapping
- Interactive OrbitControls: orbit, pan, zoom

### Teach Pendant Interface
- 6 joint sliders (±360° range each)
- Cartesian XYZ sliders with live IK
- Quick-pose presets: Home, Zero, Up (smooth 500 ms animated transitions)
- Waypoint recording with 4 node types: moveJ, moveL, Wait/Sleep, Set I/O
- Waypoint list with inline delete and reorder controls
- Path playback: Play / Pause / Resume / Stop / Restart
- Undo / Redo — 60-state snapshot history (Ctrl+Z / Ctrl+Y)

### Import / Export
- **Import:** `.urpx` (PolyScope X JSON), `.script` (URScript), `.angles` (ARTAB v2 Arduino), `.csv` (Arduino IK output)
- **Export:** `.urpx` (full PolyScope X v2 schema with safety block), `.script` (standalone URScript)

### Persistence
- Auto-save to `localStorage` on every state change
- Session restored automatically on next open

---

## Quick Start

1. Download `ur20_teach_pendant_v4.html`
2. Open it in a browser (Chrome, Firefox, Edge, or Safari)
3. That's it

No Node.js. No Python. No server. No install.

---

## File Structure

| File | Description |
|------|-------------|
| `ur20_teach_pendant_v4.html` | Complete simulator — the only file you need |
| `README.md` | This document |
| `paper.md` | Academic/technical paper describing the system |

---

## Arduino CSV Pipeline

The simulator accepts CSV files output by Arduino sketches that compute inverse kinematics and log joint angles over serial.

### CSV Format

```
# Lines starting with # are ignored (comments)
# Column order: J1, J2, J3, J4, J5, J6  (degrees, UR joint order base→wrist)
j1,j2,j3,j4,j5,j6
0.00,-90.00,0.00,-90.00,0.00,0.00
15.32,-85.10,12.44,-87.20,0.00,0.00
30.11,-79.55,24.88,-85.33,0.00,0.00
```

- Values are **degrees**, not radians
- Six columns per row, comma-separated
- Header row (`j1,j2,...`) is optional — non-numeric rows are skipped automatically
- Each row becomes one **moveJ** waypoint at default 50% speed / 50% acceleration

### End-to-End Workflow

```
Arduino sketch
     │  computes IK, logs angles over Serial
     ▼
Serial monitor saves output as .csv
     │
     ▼
UR20 Simulator  ──  Import .csv  ──►  Waypoint list populated
     │
     ├──  Play  ──►  Preview trajectory in 3D
     │
     └──  Export .urpx  ──►  Load into PolyScope X or URSim
```

---

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+Z` | Undo |
| `Ctrl+Y` | Redo |
| `Space` | Play / Pause path playback |
| `Escape` | Stop playback |

---

## Export Formats

### URPX (PolyScope X Native)

Exports a `.urpx` JSON file conforming to the PolyScope X v2 program schema. Includes:

- Full safety configuration block (required by PolyScope X loader — joint limits, tool speed/force limits, safety planes at UR20 factory defaults)
- Hierarchical node tree with `ur-move-to`, `ur-wait`, and `ur-set-io` nodes
- Motion profiles with speed, acceleration, and blend radius per node
- Embedded URScript mirror with `t=N` timed-move parameters
- Robot metadata and ISO 8601 creation timestamp

Validated against URSim 5.14. Programs load and execute without schema errors.

### URScript (`.script`)

Exports a plain-text URScript program:

```
def program():
  movej([θ1,θ2,θ3,θ4,θ5,θ6], a=1.2, v=1.05, t=0)
  movel(p[x,y,z,0,0,0], a=1.2, v=0.25)
  sleep(1.5)
  set_digital_out(0, True)
end
```

Can be loaded directly into URSim's script editor or deployed to a real controller via the UR Dashboard Server `load` command.

---

## Technical Details

### DH Parameters (UR20)

| Joint | a (m) | d (m) | α (rad) |
|-------|--------|--------|---------|
| 1 | 0.0000 | 0.2363 | π/2 |
| 2 | −0.8620 | 0.0000 | 0 |
| 3 | −0.7287 | 0.0000 | 0 |
| 4 | 0.0000 | 0.2010 | π/2 |
| 5 | 0.0000 | 0.1593 | −π/2 |
| 6 | 0.0000 | 0.1543 | 0 |

Source: Universal Robots UR20 Technical Specification.

### Inverse Kinematics Solver

| Parameter | Value |
|-----------|-------|
| Method | Levenberg-Marquardt (numerical) |
| Residual | Position-only (3 DOF) |
| Jacobian | Finite-difference, 3×6, δ = 1×10⁻⁶ rad |
| Max iterations | 200 |
| Convergence tolerance | 0.1 mm |
| Linear solver | Gaussian elimination with partial pivoting |
| Damping λ init | 1×10⁻³ |
| Damping adaptation | ×0.1 on improvement, ×10 on regression |

### Robot Specs

| Property | Value |
|----------|-------|
| Model | Universal Robots UR20 |
| DOF | 6 (all revolute) |
| Reach | 1750 mm |
| Payload | 20 kg |
| Kinematics convention | Standard DH |

### Implementation

| Property | Value |
|----------|-------|
| Language | Vanilla JavaScript ES2022 |
| 3D library | Three.js r158 (CDN via importmap) |
| External dependencies | None (beyond Three.js CDN) |
| Lines of code | ~2750 |
| Deployment | Single HTML file |

---

## Browser Compatibility

| Browser | Status |
|---------|--------|
| Chrome 90+ | Supported |
| Firefox 89+ | Supported |
| Edge 90+ | Supported |
| Safari 15+ | Supported |
| Mobile browsers | Functional (touch orbit supported via OrbitControls) |

WebGL 2.0 required. Enabled by default in all listed browsers on hardware with GPU support.

---

## License

MIT License — see below.

```
MIT License

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
