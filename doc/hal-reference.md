# HAL File Reference

This document explains every command in [`config/cia402-minas-a6bf.hal`](../config/cia402-minas-a6bf.hal).

A HAL (Hardware Abstraction Layer) file is a script executed by `halcmd` at LinuxCNC start-up. It loads real-time kernel modules, registers their functions into the servo thread, and connects signals (called *nets*) between component pins.

---

## Data-flow overview

```
EtherCAT network
    ↕  lcec driver (lcec.read-all / lcec.write-all)
    ↕
 cia402 component (cia402.N.read-all / cia402.N.write-all)
   – CiA 402 state machine
   – position / velocity scaling
    ↕
 LinuxCNC motion controller  (motion-command-handler / motion-controller)
   – trajectory planning
   – joint position loop
```

The servo thread executes these stages in a fixed order every `SERVO_PERIOD` (1 ms in this configuration). The order is critical; see the [addf section](#servo-thread-function-registration) below.

---

## Module loading

### `loadrt` — load a real-time kernel module

```hal
loadrt [KINS]KINEMATICS
```

Loads the kinematics module. `[KINS]KINEMATICS` is substituted from the INI file (value: `trivkins`). `trivkins` maps joints to axes directly: joint 0 → X, joint 1 → Y, joint 2 → Z.

```hal
loadrt [EMCMOT]EMCMOT servo_period_nsec=[EMCMOT]SERVO_PERIOD num_joints=[KINS]JOINTS
```

Loads the real-time motion controller (`motmod`).

- `servo_period_nsec` — servo loop period in nanoseconds, read from `[EMCMOT]SERVO_PERIOD` (value: `1000000` = 1 ms). **Must equal `appTimePeriod` in the EtherCAT XML file.**
- `num_joints` — number of joints, read from `[KINS]JOINTS` (value: `3`).

```hal
loadrt lcec
```

Loads the LinuxCNC EtherCAT driver (`lcec`). The hardware topology (slave list, PDO mapping) is provided by the XML configuration file loaded separately by `lcec_conf`.

```hal
loadrt cia402 count=3
```

Loads the `hal-cia402` HAL component and creates **3 instances**: `cia402.0`, `cia402.1`, `cia402.2` — one per servo drive. Change `count=` to match the number of CiA 402 drives in your system.

> **Optional PID:** The commented-out line `# loadrt pid names=x-pid,y-pid,z-pid` would load three PID controllers. PID loops are not needed when the servo drive operates in CSP (Cyclic Synchronous Position) mode and does its own position control. Enable them if you have a torque-mode drive or need additional feed-forward tuning.

### `loadusr` — load a user-space process

```hal
loadusr -W lcec_conf ethercat-conf-minas-a6bf.xml
```

Runs `lcec_conf` as a user-space process. It reads the EtherCAT XML configuration file and pushes the slave topology and PDO mappings into the EtherCAT master kernel module.

- `-W` — tells `halcmd` to **wait** until `lcec_conf` exits before continuing. This is required; `loadrt lcec` must run after the configuration is loaded.
- `ethercat-conf-minas-a6bf.xml` — path to the EtherCAT XML file. Change to your file name.

---

## Servo-thread function registration

```hal
addf lcec.read-all             servo-thread
addf cia402.0.read-all         servo-thread
addf cia402.1.read-all         servo-thread
addf cia402.2.read-all         servo-thread

addf motion-command-handler    servo-thread
addf motion-controller         servo-thread

addf cia402.0.write-all        servo-thread
addf cia402.1.write-all        servo-thread
addf cia402.2.write-all        servo-thread
addf lcec.write-all            servo-thread
```

`addf` registers a real-time function to execute in the named thread every period. The **order of `addf` calls defines the execution order** within each servo cycle.

| Step | Function | Purpose |
|------|----------|---------|
| 1 | `lcec.read-all` | Read fresh PDO data from all EtherCAT slaves — hardware input for this cycle. |
| 2–4 | `cia402.N.read-all` | Decode the CiA 402 statusword, scale actual position/velocity (encoder counts → mm), advance the CiA state machine if needed. |
| 5 | `motion-command-handler` | Process any pending commands from the G-code interpreter (new target positions, mode changes, etc.). |
| 6 | `motion-controller` | Run the trajectory planner and position loop; compute new commanded positions for each joint. |
| 7–9 | `cia402.N.write-all` | Encode the CiA 402 controlword, scale commanded position/velocity (mm → encoder counts), send to the drive. |
| 10 | `lcec.write-all` | Write the updated PDO output data to all EtherCAT slaves — hardware output for this cycle. |

> **Why order matters:** If `lcec.read-all` is placed after `motion-controller`, the motion calculation runs on one-cycle-old data, introducing a systematic delay. Incorrect ordering can cause instability or subtle tracking errors in high-speed applications.

---

## E-Stop enable signal

```hal
net emc-enable => iocontrol.0.emc-enable-in
sets emc-enable 1
```

Creates a HAL signal `emc-enable` and hard-wires it to `1` (always enabled). This **bypasses the E-Stop circuit** in HAL — LinuxCNC will not fault even if a physical E-Stop is pressed.

**In a production installation**, replace `sets emc-enable 1` with a connection to a physical E-Stop relay or safety relay output:

```hal
# Example: E-Stop button wired to a digital input (active-high when safe)
net emc-enable  my-gpio.0.in => iocontrol.0.emc-enable-in
```

---

## Per-axis configuration

The X, Y, and Z axes follow an identical pattern. Only the `cia402` instance number (`0`, `1`, `2`) and the net name prefix (`x-`, `y-`, `z-`) differ.

### `setp` — set component parameter

```hal
setp cia402.0.csp-mode 1
```

Sets the operating mode for `cia402` instance 0 (X axis).

| Value | Mode | Description |
|-------|------|-------------|
| `1` (true) | **CSP** — Cyclic Synchronous Position | The drive receives a new absolute target position every servo cycle. The drive's internal position loop closes the error. Use for precision machining. |
| `0` (false) | **CSV** — Cyclic Synchronous Velocity | The drive receives a new target velocity every servo cycle. Use for spindle-type axes or when the drive does not support CSP. |

> **This parameter is read only at start-up.** Changing it while LinuxCNC is running has no effect.

```hal
setp cia402.0.pos-scale 278
```

Sets the position scale for instance 0: **278 drive position units per millimetre**.

- Conversion applied by `cia402`: `drv-target-position = pos-cmd × pos-scale`
- Inverse applied for feedback: `pos-fb = drv-actual-position ÷ pos-scale`

**Calculating `pos-scale`:** See [`scale-calculation-guide.md`](scale-calculation-guide.md). The value depends on the drive's configured encoder resolution (object 0x6064 counts per revolution) and your ball-screw pitch or gear ratio.

---

### Drive → CiA 402 signals

These nets carry real-time status from the EtherCAT drive to the `cia402` component each servo cycle.

```hal
net x-statusword      lcec.0.0.cia-statusword  => cia402.0.statusword
net x-opmode-display  lcec.0.0.opmode-display  => cia402.0.opmode-display
net x-drv-act-pos     lcec.0.0.actual-position => cia402.0.drv-actual-position
net x-drv-act-velo    lcec.0.0.actual-velocity => cia402.0.drv-actual-velocity
```

HAL pin naming for `lcec`: `lcec.<master-idx>.<slave-idx>.<hal-pin-name>`

With `master idx="0"` and `slave idx="0"` in the XML file, the prefix is `lcec.0.0.*`. For slave `idx="1"` it would be `lcec.0.1.*`.

| Signal | lcec source | cia402 destination | CiA 402 object |
|--------|------------|-------------------|----------------|
| `x-statusword` | `lcec.0.0.cia-statusword` | `cia402.0.statusword` | 0x6041 |
| `x-opmode-display` | `lcec.0.0.opmode-display` | `cia402.0.opmode-display` | 0x6061 |
| `x-drv-act-pos` | `lcec.0.0.actual-position` | `cia402.0.drv-actual-position` | 0x6064 |
| `x-drv-act-velo` | `lcec.0.0.actual-velocity` | `cia402.0.drv-actual-velocity` | 0x606C |

---

### CiA 402 → Drive signals

These nets carry control commands from the `cia402` component to the EtherCAT drive.

```hal
net x-controlword         cia402.0.controlword        => lcec.0.0.cia-controlword
net x-modes-of-operation  cia402.0.opmode             => lcec.0.0.opmode
net x-drv-target-pos      cia402.0.drv-target-position => lcec.0.0.target-position
net x-drv-target-velo     cia402.0.drv-target-velocity => lcec.0.0.target-velocity
```

| Signal | cia402 source | lcec destination | CiA 402 object |
|--------|--------------|-----------------|----------------|
| `x-controlword` | `cia402.0.controlword` | `lcec.0.0.cia-controlword` | 0x6040 |
| `x-modes-of-operation` | `cia402.0.opmode` | `lcec.0.0.opmode` | 0x6060 |
| `x-drv-target-pos` | `cia402.0.drv-target-position` | `lcec.0.0.target-position` | 0x607A |
| `x-drv-target-velo` | `cia402.0.drv-target-velocity` | `lcec.0.0.target-velocity` | 0x60FF |

> In CSP mode, `cia402` drives `target-position` (0x607A) and sets `target-velocity` (0x60FF) to zero. In CSV mode the roles are reversed.

---

### Motion ↔ CiA 402 signals

These nets bridge LinuxCNC's internal joint controller and the `cia402` component.

```hal
net x-enable    <= joint.0.amp-enable-out  => cia402.0.enable
net x-amp-fault => joint.0.amp-fault-in    <= cia402.0.drv-fault
net x-pos-cmd   <= joint.0.motor-pos-cmd   => cia402.0.pos-cmd
net x-pos-fb    => joint.0.motor-pos-fb    <= cia402.0.pos-fb
```

HAL `net` syntax: `<=` marks the signal driver (source); `=>` marks the reader (destination). A single `net` command creates a named signal and connects one driver to one or more readers.

| Signal | Source | Destination | Description |
|--------|--------|-------------|-------------|
| `x-enable` | `joint.0.amp-enable-out` | `cia402.0.enable` | LinuxCNC enables the drive when the machine is active and no fault is present. |
| `x-amp-fault` | `cia402.0.drv-fault` | `joint.0.amp-fault-in` | A drive fault causes LinuxCNC to disable the joint and report a fault. |
| `x-pos-cmd` | `joint.0.motor-pos-cmd` | `cia402.0.pos-cmd` | Commanded joint position in machine units (mm). |
| `x-pos-fb` | `cia402.0.pos-fb` | `joint.0.motor-pos-fb` | Actual joint position in machine units (mm), converted from raw encoder counts by `pos-scale`. |

---

### Homing signals

```hal
net x-home-index <= joint.0.index-enable => cia402.0.home
```

`cia402.0.home` is a bidirectional I/O pin connected to `joint.0.index-enable`. The homing sequence proceeds as follows:

1. When the operator triggers homing, LinuxCNC sets `joint.0.index-enable` high.
2. `cia402.0.home` transitions to `true`, instructing the `cia402` component to command the drive to execute its internal homing procedure (object 0x6060 = mode 6).
3. The drive executes the configured homing method (set via SDO `0x6098` in the XML, or pre-configured in the drive).
4. Once the drive reports homing complete, `cia402.0.stat-homed` goes `true` and `cia402.0.home` is cleared — this is the event LinuxCNC uses to latch the home position.

**INI requirements for drive-internal homing:**

```ini
HOME_SEARCH_VEL = 0       ; 0 = no switch search; LinuxCNC treats current position as home.
                          ;   The cia402.N.home pin then triggers the drive's internal homing.
HOME_LATCH_VEL  = 5       ; must be non-zero to activate index-enable homing
HOME_USE_INDEX  = NO      ; the cia402 home pin emulates the index event
```

---

## CiA 402 HAL pin reference

All pins below are per-instance; replace `N` with the instance number (0, 1, 2, …). HAL pin names use hyphens where the `.comp` source uses underscores (e.g., `drv_actual_position` → `drv-actual-position`).

### Input pins (from drive, updated by `cia402.N.read-all`)

| HAL pin | Type | CiA 402 object | Description |
|---------|------|---------------|-------------|
| `cia402.N.statusword` | `u32` in | 0x6041 | Raw CiA 402 Statusword from the drive. |
| `cia402.N.opmode-display` | `s32` in | 0x6061 | Currently active mode of operation (feedback). |
| `cia402.N.drv-actual-position` | `s32` in | 0x6064 | Raw encoder position in drive counts. |
| `cia402.N.drv-actual-velocity` | `s32` in | 0x606C | Raw encoder velocity in drive units. |

### Output pins (to drive, updated by `cia402.N.write-all`)

| HAL pin | Type | CiA 402 object | Description |
|---------|------|---------------|-------------|
| `cia402.N.controlword` | `u32` out | 0x6040 | Raw CiA 402 Controlword sent to the drive. |
| `cia402.N.opmode` | `s32` out | 0x6060 | Mode of operation command (6 = homing, 8 = CSP, 9 = CSV). |
| `cia402.N.drv-target-position` | `s32` out | 0x607A | Target position in drive counts (CSP mode). |
| `cia402.N.drv-target-velocity` | `s32` out | 0x60FF | Target velocity in drive units (CSV mode). |

### Motion interface pins

| HAL pin | Type | Description |
|---------|------|-------------|
| `cia402.N.enable` | `bit` in | `true` enables the drive. Connect to `joint.N.amp-enable-out`. |
| `cia402.N.pos-cmd` | `float` in | Commanded position in machine units. Connect to `joint.N.motor-pos-cmd`. |
| `cia402.N.velocity-cmd` | `float` in | Commanded velocity in machine units/s (CSV mode or feedforward). |
| `cia402.N.pos-fb` | `float` out | Actual position in machine units. Connect to `joint.N.motor-pos-fb`. |
| `cia402.N.velocity-fb` | `float` out | Actual velocity in machine units/s. |
| `cia402.N.drv-fault` | `bit` out | `true` indicates a drive fault. Connect to `joint.N.amp-fault-in`. |
| `cia402.N.fault-reset` | `bit` in | Pulse `true` to send a fault-reset command to the drive. |

### Homing pins

| HAL pin | Type | Description |
|---------|------|-------------|
| `cia402.N.home` | `bit` I/O | `true` triggers the drive's internal homing procedure; cleared automatically on completion. Connect to `joint.N.index-enable`. |
| `cia402.N.stat-homed` | `bit` out | `true` once the drive's internal homing is complete. |
| `cia402.N.stat-homing` | `bit` out | `true` while the drive homing procedure is running. |

### CiA 402 state status pins

These decode the drive's Statusword (0x6041) into individual readable flags.

| HAL pin | Type | Description |
|---------|------|-------------|
| `cia402.N.stat-switchon-ready` | `bit` out | CiA state: Ready to Switch On. |
| `cia402.N.stat-switched-on` | `bit` out | CiA state: Switched On. |
| `cia402.N.stat-op-enabled` | `bit` out | CiA state: Operation Enabled (normal running state). |
| `cia402.N.stat-voltage-enabled` | `bit` out | CiA state: bus voltage present. |
| `cia402.N.stat-fault` | `bit` out | Drive has the fault bit set in the Statusword. |
| `cia402.N.stat-quick-stop` | `bit` out | Drive is in Quick Stop state. |
| `cia402.N.stat-switchon-disabled` | `bit` out | CiA state: Switch On Disabled (initial power-on state). |
| `cia402.N.stat-warning` | `bit` out | Drive has a warning condition (non-fatal). |
| `cia402.N.stat-remote` | `bit` out | Drive is in remote (bus-controlled) operation. |
| `cia402.N.stat-target-reached` | `bit` out | Drive reports it has reached the commanded position or velocity. |
| `cia402.N.opmode-no-mode` | `bit` out | No operating mode is active on the drive. |
| `cia402.N.opmode-homing` | `bit` out | Homing mode is active. |
| `cia402.N.opmode-cyclic-position` | `bit` out | CSP mode is active. |
| `cia402.N.opmode-cyclic-velocity` | `bit` out | CSV mode is active. |

### Parameters (`setp`)

| HAL parameter | Type | Default | Description |
|---------------|------|---------|-------------|
| `cia402.N.pos-scale` | `float` rw | `1.0` | Drive position units per machine unit (mm). The key hardware-specific tuning value. See [`scale-calculation-guide.md`](scale-calculation-guide.md). |
| `cia402.N.velo-scale` | `float` rw | `1.0` | Machine units (mm) per motor revolution, used to scale velocity feedback. Set to the ball-screw pitch (mm/rev) for a direct-drive linear axis. |
| `cia402.N.auto-fault-reset` | `bit` rw | `false` | When `true`, the component automatically sends a fault-reset command to the drive at the next rising edge of `enable`. |
| `cia402.N.csp-mode` | `bit` rw | `true` | `true` = CSP, `false` = CSV. Read at start-up only. |

---

## Cross-references

- [ini-reference.md](ini-reference.md) — machine configuration variables.
- [ethercat-xml-reference.md](ethercat-xml-reference.md) — EtherCAT slave configuration and PDO mapping.
- [scale-calculation-guide.md](scale-calculation-guide.md) — how to derive `pos-scale` and `velo-scale` from hardware specs.
