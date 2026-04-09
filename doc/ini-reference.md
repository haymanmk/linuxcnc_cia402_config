# INI File Reference

This document explains every variable in [`config/axis_mm_minas_a6bf.ini`](../config/axis_mm_minas_a6bf.ini).

The INI file is the primary configuration file passed to the `linuxcnc` command. It controls machine identity, display settings, motion limits, kinematics, and which HAL files to load. Each section is enclosed in square brackets (e.g., `[EMC]`). Comments begin with `#` or `;`.

---

## [EMC]

General machine identity and debug settings.

| Variable | Example value | Description |
|----------|--------------|-------------|
| `VERSION` | `1.1` | INI format version. Do not change; it is read by LinuxCNC to verify file compatibility. |
| `MACHINE` | `LinuxCNC-HAL-AXIS` | Human-readable name shown in the display title bar and log. Change to whatever identifies your machine (e.g., `My-Gantry-Mill`). |
| `DEBUG` | `0x0040` | Bitmask controlling the verbosity of LinuxCNC's internal log output. `0x0` disables all messages. `0x7FFFFFFF` enables everything (very verbose, for troubleshooting). `0x0040` enables NML (inter-process messaging) debug. Leave at `0x0` for normal operation. |

---

## [DISPLAY]

Settings for the graphical user interface (GUI) or headless display.

| Variable | Example value | Unit | Description |
|----------|--------------|------|-------------|
| `DISPLAY` | `linuxcncrsh` | — | The display program to launch. Common values: `axis` (standard AXIS GUI), `touchy` (touchscreen), `gmoccapy` (modern GUI), `linuxcncrsh` (remote shell, no GUI window). |
| `CYCLE_TIME` | `0.100` | s | How often the display refreshes its position readout. `0.100` (100 ms) is standard. Lower values increase CPU load; higher values make the display feel sluggish. |
| `POSITION_OFFSET` | `RELATIVE` | — | Selects whether the DRO shows position relative to the active work coordinate system (`RELATIVE`) or to the machine home position (`MACHINE`). |
| `POSITION_FEEDBACK` | `ACTUAL` | — | Selects whether the DRO shows the commanded position (`COMMANDED`) or the actual encoder feedback position (`ACTUAL`). `ACTUAL` is recommended for servo systems. |
| `MAX_FEED_OVERRIDE` | `1.2` | ratio | Maximum feed-rate override multiplier allowed in the GUI. `1.2` means the operator can raise feed rate to 120% of the programmed value. |
| `MAX_SPINDLE_OVERRIDE` | `1.0` | ratio | Maximum spindle speed override multiplier. `1.0` means no override above the programmed speed. |
| `MIN_ANGULAR_VELOCITY` | `1` | rpm | Minimum angular velocity shown in the GUI spindle speed readout. |
| `MAX_ANGULAR_VELOCITY` | `360` | rpm | Maximum angular velocity shown in the GUI spindle speed readout. |
| `PROGRAM_PREFIX` | `/home/cnc/linuxcnc/nc_files` | path | Default directory the file browser opens when loading G-code programs. Change to the directory where you store your NC files. |
| `INCREMENTS` | `1 mm, .01 in, .1mm, 1 mil, .1 mil, 1/8000 in` | — | Comma-separated list of jog step increments available in the GUI. Customize for your preferred working increments. |

**Adjusting for your hardware:** Set `DISPLAY` to `axis` for the standard GUI. Update `PROGRAM_PREFIX` to your actual NC file storage directory.

---

## [FILTER]

File-type converters that transform images or scripts into G-code before LinuxCNC runs them. Each `PROGRAM_EXTENSION` line declares a file type; the subsequent mapping lines name the command to run on matching files.

| Variable | Example value | Description |
|----------|--------------|-------------|
| `PROGRAM_EXTENSION = .png,.gif,.jpg …` | `Grayscale Depth Image` | Associates `.png`, `.gif`, and `.jpg` extensions with the description "Grayscale Depth Image". |
| `PROGRAM_EXTENSION = .py …` | `Python Script` | Associates `.py` with the description "Python Script". |
| `png = image-to-gcode` | — | Files with `.png` are piped through the `image-to-gcode` utility before LinuxCNC executes them. |
| `gif = image-to-gcode` | — | Same for `.gif`. |
| `jpg = image-to-gcode` | — | Same for `.jpg`. |
| `py = python3` | — | Python scripts are executed with `python3`; their stdout output is fed to LinuxCNC as G-code. |

**Adjusting for your hardware:** This section is independent of hardware. Add your own converters for custom CAM post-processors or other file types.

---

## [TASK]

Settings for the LinuxCNC task controller, which coordinates the interpreter, motion planner, and I/O.

| Variable | Example value | Unit | Description |
|----------|--------------|------|-------------|
| `TASK` | `milltask` | — | The task controller program to use. `milltask` is the standard milling-machine task controller. Do not change unless using a custom task controller. |
| `CYCLE_TIME` | `0.001` | s | How frequently the task controller polls for new commands and updates state (1 ms). Very rarely needs adjustment. |

---

## [RS274NGC]

Settings for the G-code interpreter (RS-274/NGC dialect).

| Variable | Example value | Description |
|----------|--------------|-------------|
| `PARAMETER_FILE` | `sim_mm.var` | Path to the file where the interpreter stores persistent G-code variables (e.g., `#5221` work offset, `#1`–`#30` global named parameters). The file is created automatically if it does not exist. Name it something meaningful for your machine (e.g., `my-mill.var`). |

---

## [EMCMOT]

Settings for the real-time motion controller.

| Variable | Example value | Unit | Description |
|----------|--------------|------|-------------|
| `EMCMOT` | `motmod` | — | The motion controller module to load. `motmod` is always correct for standard LinuxCNC. |
| `COMM_TIMEOUT` | `1.0` | s | How long LinuxCNC waits for a response from the motion controller before declaring a communication error. `1.0` s is the standard value. |
| `BASE_PERIOD` | `0` | ns | The base thread period. Set to `0` when not using a base thread (e.g., no step-generation hardware that requires a dedicated fast thread). EtherCAT/CiA 402 systems do not require a base thread. |
| `SERVO_PERIOD` | `1000000` | ns | The servo thread period. **This value must equal `appTimePeriod` in the EtherCAT XML file.** `1000000` ns = 1 ms = 1 kHz servo loop. Common alternatives: `2000000` (500 Hz). Reducing below 1 ms increases CPU load and may cause real-time overruns. |

**Adjusting for your hardware:** `SERVO_PERIOD` and `appTimePeriod` (EtherCAT XML) must always match. If you change one, change the other. See [`scale-calculation-guide.md`](scale-calculation-guide.md) for the verification step.

---

## [EMCIO]

Settings for the I/O controller (handles tool changes, coolant, E-Stop, etc.).

| Variable | Example value | Description |
|----------|--------------|-------------|
| `EMCIO` | `io` | The I/O controller program. `io` is the standard controller. |
| `CYCLE_TIME` | `0.100` | How often the I/O controller polls for updates (100 ms). Rarely needs adjustment. |
| `TOOL_TABLE` | `sim_mm.tbl` | Path to the tool table file, which stores tool lengths and diameters for tool-length compensation. Change to the actual path of your tool table file. |
| `TOOL_CHANGE_POSITION` | `0 0 50.8` | Machine coordinates (X Y Z in machine units) that the axes move to before a tool change. Set to a safe, clear position well above the workpiece. |

---

## [HAL]

Specifies which HAL files to load and optional HAL UI components.

| Variable | Example value | Description |
|----------|--------------|-------------|
| `HALFILE` | `cia402-minas-a6bf.hal` | Path to a HAL script executed at startup. Multiple `HALFILE` lines are allowed; they run in the order listed. This is where hardware connections are established. Change to the name of your HAL file. |
| `HALUI` | `halui` | Loads the `halui` component, which exposes HAL pins for basic machine control (useful for jog buttons, MPG pendants, etc.). |
| `POSTGUI_HALFILE` | *(commented out)* | A HAL file executed after the GUI starts. Used when the GUI creates its own HAL component (e.g., the AXIS GUI). The main `HALFILE` entries run before the GUI. |
| `HALCMD` | *(commented out)* | Individual `halcmd` commands to execute after all `HALFILE` scripts. |

---

## [TRAJ]

Global trajectory planner settings. These act as **ceiling values** — effective limits are set per-axis in `[AXIS_*]` and per-joint in `[JOINT_*]`.

| Variable | Example value | Unit | Description |
|----------|--------------|------|-------------|
| `COORDINATES` | `X Y Z` | — | The names of the axes as seen by the G-code interpreter. `X Y Z` is standard for a 3-axis Cartesian machine. Add `A`, `B`, `C` for rotary axes. |
| `LINEAR_UNITS` | `mm` | — | The machine's linear distance unit. `mm` or `inch`. **All position and velocity limits in `[AXIS_*]` and `[JOINT_*]` must use this unit consistently.** |
| `ANGULAR_UNITS` | `degree` | — | The machine's angular unit. `degree` or `radian`. |
| `DEFAULT_LINEAR_VELOCITY` | `30.48` | mm/s | Default G0 rapid velocity when the operator has not set a feed rate override. |
| `MAX_LINEAR_VELOCITY` | `1080000` | mm/s | Absolute ceiling velocity for the trajectory planner. Set equal to or above the highest `[AXIS_*] MAX_VELOCITY`. Setting it very high defers effective limiting to the per-axis settings. |
| `MAX_VELOCITY` | `1080000` | mm/s | Alias for `MAX_LINEAR_VELOCITY`. Both can coexist; set them consistently. |
| `DEFAULT_LINEAR_ACCELERATION` | `508` | mm/s² | Default acceleration when not otherwise specified. |
| `MAX_LINEAR_ACCELERATION` | `100000` | mm/s² | Absolute ceiling acceleration. Set equal to or above the highest `[AXIS_*] MAX_ACCELERATION`. |

---

## [KINS]

Kinematics configuration.

| Variable | Example value | Description |
|----------|--------------|-------------|
| `KINEMATICS` | `trivkins` | The kinematics module. `trivkins` maps each joint directly to the corresponding axis (joint 0 = X, joint 1 = Y, joint 2 = Z). Use `trivkins coordinates=XYZA` for an additional rotary axis, or a different module (e.g., `genhexkins`, `xyzbc-trt-kins`) for non-Cartesian machines. |
| `JOINTS` | `3` | The number of physical actuators (joints). Must match the number of `[JOINT_N]` sections and the `num_joints` argument to `motmod` in the HAL file. For a 4-axis machine, set `JOINTS = 4`. |

---

## [AXIS_X], [AXIS_Y], [AXIS_Z]

Axis-level soft limits and velocity/acceleration ceilings, in **machine units (mm)**, as seen by the G-code interpreter.

> **Axis vs. Joint:** An *axis* is a Cartesian coordinate seen by G-code (X, Y, Z). A *joint* is a physical actuator. With `trivkins`, axis X corresponds to joint 0, axis Y to joint 1, axis Z to joint 2.

| Variable | AXIS_X | AXIS_Y | AXIS_Z | Unit | Description |
|----------|--------|--------|--------|------|-------------|
| `TYPE` | `LINEAR` | *(same)* | *(same)* | — | `LINEAR` for a translational axis; `ANGULAR` for a rotary axis. |
| `MAX_VELOCITY` | `18000` | `30.48` | `30.48` | mm/s | Maximum G0 rapid velocity along this axis. The trajectory planner enforces this limit. |
| `MAX_ACCELERATION` | `90000` | `508` | `508` | mm/s² | Maximum acceleration. Exceeding the drive's peak torque capability will cause position errors or faults; start conservatively. |
| `MIN_LIMIT` | `-36000` | `-254` | `-50.8` | mm | Soft lower limit. LinuxCNC refuses to command below this position. Set just inside the physical travel limit (hard stop or limit switch position). |
| `MAX_LIMIT` | `36000` | `254` | `101.6` | mm | Soft upper limit. |

**Adjusting for your hardware:**

- Measure the actual travel range of your stage with a dial gauge or by jogging to each end stop.
- Set `MIN_LIMIT` and `MAX_LIMIT` a few mm inside the hard stop positions to provide a safety margin.
- Set `MAX_VELOCITY` to 80–90% of the maximum speed your drive, motor, and mechanical system can sustain without position error.

---

## [JOINT_0], [JOINT_1], [JOINT_2]

Per-joint settings for each physical actuator. These configure the motion controller's position loop, following-error tolerance, and homing behaviour.

> **Scale note:** When using the `cia402` HAL component, the `pos-scale` HAL parameter on the `cia402` instance handles encoder-count ↔ machine-unit conversion *before* values reach the `joint.N` HAL pins. The `motor-pos-cmd` and `motor-pos-fb` joint pins therefore carry values in machine units (mm). See [`hal-reference.md`](hal-reference.md) and [`scale-calculation-guide.md`](scale-calculation-guide.md) for details.

### Motion parameters

| Variable | JOINT_0 | JOINT_1 | JOINT_2 | Unit | Description |
|----------|---------|---------|---------|------|-------------|
| `TYPE` | `LINEAR` | `LINEAR` | `LINEAR` | — | `LINEAR` or `ANGULAR`. |
| `HOME` | `0.000` | `0.000` | `0.000` | mm | Machine coordinate assigned to the joint after homing completes. Typically `0`. |
| `MAX_VELOCITY` | `18000` | `30.48` | `30.48` | mm/s | Maximum joint velocity. Must not exceed the corresponding `[AXIS_*] MAX_VELOCITY`. |
| `MAX_ACCELERATION` | `9000` | `508` | `508` | mm/s² | Maximum joint acceleration. |
| `BACKLASH` | `0.000` | `0.000` | `0.000` | mm | Software backlash compensation. Non-zero values add a correction when direction reverses. **Not recommended for servo drives; address mechanical backlash physically instead.** |
| `MIN_LIMIT` | `-36000` | `-254` | `-50.8` | mm | Joint soft lower limit (same meaning as the corresponding `[AXIS_*]` value). |
| `MAX_LIMIT` | `36000` | `254` | `101.6` | mm | Joint soft upper limit. |

### Following error limits

| Variable | JOINT_0 | JOINT_1 | JOINT_2 | Unit | Description |
|----------|---------|---------|---------|------|-------------|
| `FERROR` | `2000.0` | `1.27` | `1.27` | mm | Maximum allowed following error at full speed. If the difference between commanded and actual position exceeds this, LinuxCNC triggers a fault and stops motion. |
| `MIN_FERROR` | `1000.0` | `.254` | `.254` | mm | Maximum allowed following error at zero velocity. The actual limit is linearly interpolated between `MIN_FERROR` (at v = 0) and `FERROR` (at `MAX_VELOCITY`). |

**Rule of thumb:** `FERROR` ≥ `MAX_VELOCITY × SERVO_PERIOD × 2`. For a 400 mm/s axis at 1 ms servo period: FERROR ≥ 0.8 mm. See [`scale-calculation-guide.md`](scale-calculation-guide.md).

### Scale parameters

| Variable | JOINT_0 | JOINT_1 | JOINT_2 | Unit | Description |
|----------|---------|---------|---------|------|-------------|
| `SCALE` | `23302` | *(not set)* | *(not set)* | counts/mm | Output scale — encoder counts per machine unit. Used by LinuxCNC for velocity feedforward calculations. When the `cia402` component fully handles scaling, set this to `1` unless you need feedforward tuning. |
| `INPUT_SCALE` | `1.000` | `157.48` | `157.48` | counts/mm | Feedback scale — encoder counts per machine unit for the position feedback signal. In a `cia402`-based system where `pos-fb` is already in mm, set to `1.0`. |
| `OUTPUT_SCALE` | *(not set)* | `1.000` | `1.000` | ratio | Additional output multiplier. `1.0` means no additional scaling. |

**Deriving scale values:** See [`scale-calculation-guide.md`](scale-calculation-guide.md) for the step-by-step calculation from encoder resolution and mechanical drive ratio.

### Homing parameters

| Variable | JOINT_0 | JOINT_1 | JOINT_2 | Unit | Description |
|----------|---------|---------|---------|------|-------------|
| `HOME_OFFSET` | `0.0` | `0.0` | `0.0` | mm | Offset added to the home switch trigger position. Use a non-zero value if the physical home switch is not at the exact desired zero point. |
| `HOME_SEARCH_VEL` | `0` | `0` | `0` | mm/s | Velocity for the initial search toward the home switch. `0` disables home-switch searching; the drive's internal homing procedure is used instead (triggered via `cia402.N.home`). A value of zero also means assume that the current location is the home position for the machine. |
| `HOME_LATCH_VEL` | `0` | `0` | `0` | mm/s | Velocity for the slow approach after the home switch is found. Set to a non-zero value when `HOME_USE_INDEX = YES` or when using drive-internal homing (any non-zero value; the drive controls the speed). |
| `HOME_USE_INDEX` | `NO` | `NO` | `NO` | — | `YES` to use the encoder index pulse for precision home latching. When `YES`, also set `HOME_LATCH_VEL` to a non-zero value. |
| `HOME_IGNORE_LIMITS` | `NO` | `NO` | `NO` | — | `YES` allows the homing procedure to travel past a software limit switch. Leave `NO` unless the home switch is outside the normal travel range. |
| `HOME_SEQUENCE` | `1` | `1` | `0` | integer | Homing order when "Home All" is invoked. Joints home in ascending order of sequence number; joints sharing the **same positive number** home simultaneously and independently — each makes its final move to `HOME` as soon as it is ready. **Negative values** (`-1`, `-2`, …) also group joints, but add synchronization: all joints in the group hold their final move to `HOME` until every joint in the group has completed its search/latch phase, then they all move to `HOME` simultaneously. The ordering uses the **absolute value** (`|-1|` = 1 comes after 0). Example: Z at `0` homes first alone; X and Y at `1` (positive) then home simultaneously but independently; or X and Y at `-1` (negative) would home simultaneously with a synchronized final HOME move. |
| `HOME_IS_SHARED` | `1` | `1` | `1` | 0 or 1 | `1` indicates the home and limit switch inputs share the same hardware signal. Only meaningful when `HOME_SEARCH_VEL ≠ 0`. |

---

## Cross-references

- [hal-reference.md](hal-reference.md) — explains the HAL script that wires hardware to LinuxCNC.
- [ethercat-xml-reference.md](ethercat-xml-reference.md) — explains the EtherCAT slave and PDO configuration.
- [scale-calculation-guide.md](scale-calculation-guide.md) — step-by-step derivation of `SCALE`, `INPUT_SCALE`, and `pos-scale` from hardware specifications.
