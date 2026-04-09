# Scale Calculation Guide

This guide explains how to derive the scale-related configuration values from your motor, encoder, and mechanical specifications. Incorrect scale values are among the most common configuration errors in LinuxCNC, leading to wrong movement distances, velocity faults, or following errors.

---

## The scale chain

When LinuxCNC commands a joint to move, the position value passes through several layers of conversion before the drive acts on it:

```
LinuxCNC motion controller
        │
        │  motor-pos-cmd  [machine units, e.g. mm]
        ▼
  joint.N.motor-pos-cmd  HAL pin
        │
        │  pos-cmd  [machine units, mm]
        ▼
  cia402.N component
        │
        │  × pos-scale  (counts/mm)
        ▼
  drv-target-position  [drive encoder counts]
        │
        │  EtherCAT PDO 0x607A
        ▼
  Servo drive internal position loop
```

The reverse path (feedback) is:

```
  Servo drive encoder
        │
        │  EtherCAT PDO 0x6064
        ▼
  actual-position  [drive encoder counts]
        │
        │  ÷ pos-scale  (counts/mm)
        ▼
  cia402.N component  →  pos-fb  [machine units, mm]
        │
        ▼
  joint.N.motor-pos-fb  HAL pin
        │
        ▼
  LinuxCNC motion controller
```

The three values you need to set correctly are:

| Parameter | Location | Meaning |
|-----------|----------|---------|
| `pos-scale` | HAL `setp` | Drive encoder counts per machine unit (mm) |
| `SCALE` | INI `[JOINT_N]` | Same physical quantity; used by the motion planner for internal velocity/acceleration calculations |
| `INPUT_SCALE` | INI `[JOINT_N]` | Feedback scale (counts/mm); when `cia402` handles all scaling, set to `1.0` |

---

## Step 1 — Determine encoder resolution

The encoder resolution is the number of **counts per motor revolution** as reported by the drive over EtherCAT (object 0x6064).

**Check your drive manual** for the encoder bit depth:

| Encoder type | Counts per revolution |
|-------------|----------------------|
| 17-bit | 131,072 |
| 20-bit | 1,048,576 |
| 23-bit absolute (MINAS-A6BF) | 8,388,608 |

> **Important — electronic gearing:** Many drives allow configuring the resolution of the position value they report in PDO 0x6064. On the Panasonic MINAS-A6BF this is controlled by the `Pr0.09` parameter (position command pulse divider / encoder output denominator). If electronic gearing is active, the counts in object 0x6064 may be a fraction of the raw encoder resolution.
>
> **Always verify empirically:** command the drive to rotate exactly one revolution (using the drive's setup software or a manual jog command), and read the change in object 0x6064. That difference is your *effective* counts per revolution.

---

## Step 2 — Determine the mechanical drive ratio

The mechanical configuration converts motor revolutions to linear displacement (mm).

### Ball-screw drive (direct drive)

```
distance_per_rev [mm/rev] = ball-screw pitch [mm/rev]
```

### Ball-screw with external gearbox

```
distance_per_rev [mm/rev] = ball-screw pitch [mm/rev] ÷ gearbox ratio
```

A gearbox ratio of 3:1 (motor spins 3× for 1 output revolution) means:

```
distance_per_rev = pitch ÷ 3
```

### Rack-and-pinion drive

```
distance_per_rev [mm/rev] = π × pinion pitch diameter [mm]
```

### Linear motor with linear encoder

The drive reports actual linear position in encoder line counts. The encoder line pitch (e.g., 20 µm per line × 4× interpolation = 5 µm per count = 0.005 mm/count) determines the scale directly:

```
pos-scale = 1 ÷ 0.005 mm/count = 200 counts/mm
```

---

## Step 3 — Calculate `pos-scale`

```
pos-scale [counts/mm] = effective_counts_per_rev ÷ distance_per_rev [mm/rev]
```

### Example

| Parameter | Value |
|-----------|-------|
| Encoder (after electronic gearing) | 131,072 counts/rev |
| Ball-screw pitch | 10 mm/rev |
| Gearbox ratio | 1:1 (direct drive) |
| distance_per_rev | 10 mm/rev |
| **pos-scale** | 131,072 ÷ 10 = **13,107.2 counts/mm** |

Apply in the HAL file:

```hal
setp cia402.0.pos-scale 13107.2
```

Fractional values are allowed: `pos-scale` is a floating-point parameter.

---

## Step 4 — Set `SCALE` and `INPUT_SCALE` in `[JOINT_N]`

With the `cia402` component handling the encoder-count ↔ machine-unit conversion, the `motor-pos-cmd` and `motor-pos-fb` HAL pins already carry values in machine units (mm). In a basic configuration you therefore set:

```ini
[JOINT_0]
INPUT_SCALE = 1.0     ; pos-fb is already in mm — no further scaling needed
SCALE       = 1.0     ; pos-cmd is already in mm
```

If you need LinuxCNC's internal velocity-feedforward calculations to use the true encoder resolution (for advanced servo tuning), set `SCALE` to match `pos-scale`:

```ini
[JOINT_0]
SCALE = 13107.2       ; must match:  setp cia402.0.pos-scale 13107.2
```

> **Regarding the example file:** `[JOINT_0]` uses `SCALE = 23302` and `[JOINT_1]` / `[JOINT_2]` use `INPUT_SCALE = 157.48`. These values reflect the specific hardware tuning (electronic gearing and ball-screw pitch) of the target machine and are not transferable to other systems without re-derivation.

---

## Step 5 — Calculate velocity and acceleration limits

All velocity values in LinuxCNC `[AXIS_*]` and `[JOINT_*]` sections are in **machine units per second** (mm/s for a metric machine).

### `MAX_VELOCITY`

```
MAX_VELOCITY [mm/s] = motor_rated_max_speed [rpm] × distance_per_rev [mm/rev] ÷ 60
```

Apply a safety derating factor (typically 0.8) to leave headroom:

```
MAX_VELOCITY [mm/s] = rated_rpm × pitch_mm × 0.8 ÷ 60
```

**Example** (3,000 rpm motor, 10 mm ball-screw pitch, 0.8 derating):

```
MAX_VELOCITY = 3000 × 10 × 0.8 ÷ 60 = 400 mm/s
```

```ini
[AXIS_X]
MAX_VELOCITY = 400

[JOINT_0]
MAX_VELOCITY = 400
```

> Set the same value in both `[AXIS_N]` and `[JOINT_N]`. The `[TRAJ] MAX_LINEAR_VELOCITY` must be at least as large as the largest `[AXIS_*]` value (or set it to a very large number to let axis limits govern).

### `MAX_ACCELERATION`

A practical starting value is to achieve maximum velocity in 0.1–0.2 seconds:

```
MAX_ACCELERATION [mm/s²] = MAX_VELOCITY [mm/s] ÷ ramp_time [s]
```

**Example** (400 mm/s, 0.1 s ramp):

```
MAX_ACCELERATION = 400 ÷ 0.1 = 4000 mm/s²
```

Start conservatively (longer ramp time = lower acceleration). Increase only after verifying the drive does not fault on acceleration transients.

---

## Step 6 — Set `FERROR` and `MIN_FERROR`

Following error limits define the maximum permissible lag between commanded and actual position before LinuxCNC faults.

### Rule of thumb

```
FERROR     ≥ MAX_VELOCITY [mm/s] × SERVO_PERIOD [s] × 2
MIN_FERROR ≥ MAX_VELOCITY [mm/s] × SERVO_PERIOD [s] × 0.5
```

For 400 mm/s and a 1 ms servo period:

```
FERROR     ≥ 400 × 0.001 × 2 = 0.8 mm   →  use 1.0–2.0 mm
MIN_FERROR ≥ 400 × 0.001 × 0.5 = 0.2 mm →  use 0.1–0.5 mm
```

Set `FERROR` large enough to avoid nuisance trips during normal acceleration, but small enough to catch genuine mechanical or drive failures promptly.

---

## Step 7 — Verify `appTimePeriod` matches `SERVO_PERIOD`

These two values must always be identical. Both are in nanoseconds.

| File | Key | Example |
|------|-----|---------|
| `ethercat-conf-*.xml` | `<master appTimePeriod="...">` | `1000000` |
| `axis_mm_*.ini` | `[EMCMOT] SERVO_PERIOD` | `1000000` |

If you change the servo loop rate (e.g., to 500 Hz = 2 ms), update **both**:

```xml
<!-- ethercat XML -->
<master idx="0" appTimePeriod="2000000" refClockSyncCycles="1">
```

```ini
; INI file
[EMCMOT]
SERVO_PERIOD = 2000000
```

---

## Worked example: Panasonic MINAS-A6BF on a ball-screw linear stage

### Hardware

| Parameter | Value |
|-----------|-------|
| Drive | Panasonic MINAS-A6BF (e.g. MBDLT25BF, 250 W) |
| Motor encoder | 23-bit multi-turn absolute, 8,388,608 counts/revolution |
| Electronic gearing (`Pr0.09`) | Set so PDO 0x6064 reports 131,072 counts/rev |
| Ball-screw pitch | 10 mm/rev |
| Gearbox | None (direct drive) |
| Motor rated speed | 3,000 rpm |
| Servo loop period | 1 ms |

### Calculations

```
effective_counts_per_rev = 131,072   (verified empirically via object 0x6064)
distance_per_rev         = 10 mm/rev (ball-screw pitch, no gearbox)

pos-scale = 131,072 ÷ 10 = 13,107.2 counts/mm

MAX_VELOCITY    = 3000 × 10 × 0.8 ÷ 60 = 400 mm/s
MAX_ACCELERATION = 400 ÷ 0.1 = 4,000 mm/s²

FERROR     = 400 × 0.001 × 3 = 1.2 mm   (3 servo cycles of margin)
MIN_FERROR = 0.1 mm
```

### Configuration snippets

**HAL file:**

```hal
setp cia402.0.csp-mode  1
setp cia402.0.pos-scale 13107.2
setp cia402.0.velo-scale 10      ; mm per motor revolution = ball-screw pitch
```

**INI `[AXIS_X]`:**

```ini
[AXIS_X]
TYPE             = LINEAR
MAX_VELOCITY     = 400
MAX_ACCELERATION = 4000
MIN_LIMIT        = -500
MAX_LIMIT        =  500
```

**INI `[JOINT_0]`:**

```ini
[JOINT_0]
TYPE             = LINEAR
HOME             = 0.000
MAX_VELOCITY     = 400
MAX_ACCELERATION = 4000
INPUT_SCALE      = 1.0
SCALE            = 13107.2
MIN_LIMIT        = -500
MAX_LIMIT        =  500
FERROR           = 1.2
MIN_FERROR       = 0.1
HOME_SEARCH_VEL  = 0    ; 0 = treat current position as home (no switch search)
HOME_LATCH_VEL   = 20
HOME_USE_INDEX   = NO
HOME_SEQUENCE    = 1
HOME_IS_SHARED   = 1
```

**EtherCAT XML `<master>`:**

```xml
<master idx="0" appTimePeriod="1000000" refClockSyncCycles="1">
```

**INI `[EMCMOT]`:**

```ini
[EMCMOT]
SERVO_PERIOD = 1000000
```

---

## Configuration checklist

- [ ] Determine effective counts/rev: command one full motor revolution and read the delta in object 0x6064.
- [ ] Measure ball-screw pitch (or other drive ratio) from the mechanical drawings or by measuring.
- [ ] Calculate `pos-scale` = effective_counts_per_rev ÷ distance_per_rev.
- [ ] Apply `setp cia402.N.pos-scale <value>` in the HAL file.
- [ ] Set `SCALE` in `[JOINT_N]` consistent with `pos-scale` (or `1.0` if feedforward tuning is not needed).
- [ ] Set `MAX_VELOCITY` (mm/s) from motor rated speed and mechanics; apply it to both `[AXIS_N]` and `[JOINT_N]`.
- [ ] Set `MAX_ACCELERATION` conservatively; increase after successful commissioning.
- [ ] Set `FERROR` ≥ `MAX_VELOCITY × SERVO_PERIOD × 2`.
- [ ] Confirm `appTimePeriod` in the EtherCAT XML equals `SERVO_PERIOD` in the INI.
- [ ] **Verify:** jog the axis exactly 100 mm and measure with a dial gauge or linear scale. Actual distance should match commanded distance within encoder resolution.

---

## Cross-references

- [ini-reference.md](ini-reference.md) — `SCALE`, `INPUT_SCALE`, `FERROR`, `MAX_VELOCITY`.
- [hal-reference.md](hal-reference.md) — `pos-scale` and `velo-scale` parameters.
- [ethercat-xml-reference.md](ethercat-xml-reference.md) — `appTimePeriod`, PDO 0x6064 resolution.
