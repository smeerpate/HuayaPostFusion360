# Huaya CNC — Fusion 360 Post Processor (Mitsubishi M80, 4-axis A indexing)

> Fusion 360 post processor for the **Huaya CNC machining centre** running a **Mitsubishi M80 controller**, with built-in automatic 4th-axis (A-axis) indexing based on setup naming conventions.

**Keywords:** Fusion 360 post processor, Mitsubishi M80, Mitsubishi M80W, Huaya CNC, 4-axis CNC, 3+1 axis, A-axis indexing, rotary table, CNC post processor, CAM post processor, G-code, Autodesk Fusion, milling post processor, 4th axis, rotary axis, indexing table, A-axis brake, M45 M46, trunnion, rotary fixture, 4-axis milling, multi-axis CAM, CNC automation, Fusion 360 .cps, post processor customization, CNC turning table, indexing fixture, A-axis rotary, Mitsubishi CNC, M80 controller, Fusion CAM, rotary indexing, 4-axis post, axis brake clamp

---

## Overview

This post processor extends the standard Autodesk Mitsubishi M80 milling post with automatic A-axis rotary table indexing. Instead of manually inserting axis moves into your NC program, you encode the required A-axis angle directly in the **Fusion 360 setup name**. The post reads the angle, generates the brake release, rapid move, and brake clamp sequence at the correct location in the program, and suppresses any duplicate axis moves from the standard workplane logic.

**What it does automatically:**
- Parses the A-axis angle from each Fusion 360 setup name
- Outputs `M45` (brake release) → `G0 A<angle>` → `M46` (brake clamp) on every setup change
- Retracts Z to a safe height before any rotary move
- Skips the move if the angle is unchanged between consecutive setups
- Writes an A-axis indexing summary as comments at the top of the NC file
- Warns (in the NC file and the Fusion Warnings tab) when a setup name does not follow the convention

**Machine configuration:**

| Parameter | Value |
|---|---|
| Controller | Mitsubishi M80 / M80W |
| Axes | X, Y, Z + A (table rotation around X-axis) |
| A-axis range | ±360° |
| Machining mode | 3+1 indexing (no simultaneous 4-axis motion) |
| Tool length compensation | Manual — operator sets H-offsets in controller tool table |
| G68.2 tilted workplane | **Not used** (option not licensed on this machine) |
| Tool preloading | Disabled by default |

---

## Installation

1. Download `huaya_m80.cps`
2. Copy the file to your local Fusion 360 post library folder:
   - **Windows:** `%AppData%\Autodesk\Fusion 360 CAM\Posts`
   - **macOS:** `~/Library/Application Support/Autodesk/Fusion 360 CAM/Posts`
3. In Fusion 360, go to **Manufacturing** → **Post Process**
4. Select **Huaya M80 (4-axis A indexing)** as the post processor

---

## A-axis indexing — setup naming convention

The post reads the A-axis position directly from the **Fusion 360 setup name** (the name you give each setup in the CAM browser). Create one setup per A-axis position.

### Name format

```
<setup-name>_<degrees>deg
```

### Examples

| Setup name | A-axis position |
|---|---|
| `OP1_0deg` | A = 0° |
| `OP2_90deg` | A = 90° |
| `Side3_-45deg` | A = −45° |
| `Face4_22.5deg` | A = 22.5° |
| `Fixture_180deg` | A = 180° |

Rules:
- The name must end with `_<number>deg` (case-insensitive, so `DEG` and `Deg` also work)
- Negative angles are supported: `_-90deg`
- Decimal angles are supported: `_22.5deg`
- The part of the name before the underscore is free to choose

If a setup name does not follow the convention, the post writes a warning comment into the NC file and an entry in the Fusion 360 Warnings tab, and skips the A-axis move for that setup.

---

## Generated G-code

### On a setup change (new A-axis angle)

```gcode
( --- SETUP: OP2_90deg --- )
( A-AXIS INDEXING: 90 deg )
G28 G91 Z0.             ; Z retract before rotation
G90
M45                     ; Brake RELEASE
G00 A90.                ; Rapid to target angle
M46                     ; Brake CLAMP
```

### Same angle as previous setup (no move needed)

```gcode
( --- SETUP: OP2b_90deg --- )
( A-AXIS: 90 deg (no move required, already in position) )
```

### NC file header summary

At the start of every post run, the processor writes a summary of all setups and their A-axis angles:

```gcode
( ==================================== )
( A-AXIS INDEXING OVERVIEW             )
( Setup naming convention: <name>_<degrees>deg )
( e.g. OP1_0deg  /  Side2_90deg  /  Face3_-45deg )
( ------------------------------------ )
( SETUP 1: OP1_0deg  =>  A = 0 deg  [OK] )
( SETUP 2: OP2_90deg  =>  A = 90 deg  [OK] )
( ==================================== )
```

---

## Code structure

```
huaya_m80.cps
│
├── Properties
│   ├── preloadTool              Off by default (Huaya ATC does not require preloading)
│   ├── showSequenceNumbers      Block number output on/off
│   ├── sequenceNumberStart      Start number (default 10)
│   ├── sequenceNumberIncrement  Step size (default 5)
│   └── ... (standard Mitsubishi options)
│
├── Custom module-level variables
│   ├── currentNamedAAxisAngleDeg  Active A-axis angle (degrees) from setup name
│   └── currentJobDescription      Active setup name (job-description parameter)
│
├── Custom functions
│   ├── getAAxisAngleFromSetupName(setupName)
│   │     Parses the _<degrees>deg suffix from a setup name.
│   │     Returns a Number, or undefined when the pattern is absent.
│   │     Case-insensitive; supports negative and decimal values.
│   │     No regex — compatible with the Fusion post engine's JS environment.
│   │
│   └── reportAAxisIndexingSummary()
│         Scans all sections, collects unique setup names, and writes
│         the A-axis plan as NC comments in the program header.
│         Called from onOpen().
│
├── Custom logic in onSection()
│   └── Detects setup changes via the job-description parameter.
│       On a new A-axis angle: Z retract → M45 → G0 A<angle> → M46
│       Sets currentWorkPlaneABC to prevent setWorkPlane() from
│       generating duplicate axis moves.
│
├── Custom logic in setWorkPlane()
│   └── When a named A-axis angle is active, replaces the incoming
│       abc.x with the named angle before the workplane comparison.
│       Prevents the standard workplane logic from outputting an
│       unwanted move to A0. for sections programmed in the local
│       (already-rotated) coordinate frame of their setup.
│
└── Custom logic in forceWorkPlane()
      Preserves currentWorkPlaneABC across tool changes within the
      same setup, preventing setWorkPlane() from re-indexing the
      A-axis on every tool change.
```

---

## M-codes — A-axis brake control

| Code | Function |
|---|---|
| `M45` | A-axis brake **RELEASED** (before rotation) |
| `M46` | A-axis brake **CLAMPED** (after positioning) |

These codes are specific to this machine's parameter settings. Verify that M45 and M46 match the brake control configuration in your controller before running.

---

## Pre-flight checklist

Run through these checks before executing any program:

1. **Tool length offsets** — H-offsets must be set manually in the controller tool table before running. The post outputs `G43 H<tool number>`.
2. **A-axis brake codes** — confirm that M45 releases the brake and M46 clamps it on your specific machine (parameter-dependent).
3. **Z clearance before rotation** — verify that the Z retract height provides sufficient clearance for the workpiece to rotate freely.
4. **First run** — always run in **single-block mode** with low rapid override and low feedrate override until the program is verified on your machine.

---

## Full NC program structure

```gcode
%
O1001
( Huaya M80 — program header )
( ==================================== )
( A-AXIS INDEXING OVERVIEW             )
( SETUP 1: OP1_0deg  =>  A = 0 deg  [OK] )
( SETUP 2: OP2_90deg  =>  A = 90 deg  [OK] )
( ==================================== )

( --- SETUP: OP1_0deg --- )
( A-AXIS INDEXING: 0 deg )
G28 G91 Z0.
G90
M45
G00 A0.
M46

T1 M06
G17 G90 G94
G54
G43 H01 Z100.
...

( --- SETUP: OP2_90deg --- )
( A-AXIS INDEXING: 90 deg )
G28 G91 Z0.
G90
M45
G00 A90.
M46

T2 M06
G17 G90 G94
G54
G43 H02 Z100.
...

M30
%
```

---

## Based on

Autodesk Mitsubishi mill post processor, Revision 44227.  
Customised for the Huaya CNC machining centre with A-axis rotary table indexing.

> Original base: Copyright (C) 2012–2026 Autodesk, Inc.
