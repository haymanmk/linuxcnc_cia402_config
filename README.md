# Configure LinuxCNC for CiA 402 Motors

[LinuxCNC](https://linuxcnc.org/) does not support **CiA 402** protocol by default. In order to make CiA 402 viable on LinuxCNC, we have to include a few third-party tools such as [linuxcnc-ethercat](https://github.com/linuxcnc-ethercat/linuxcnc-ethercat) and [hal-cia402](https://github.com/dbraun1981/hal-cia402). This document does not cover the scope of how to install these tools on LinuxCNC but mainly focus on how to set up those complex variables in the configuration files, LinuxCNC consumes them and adapts for your hardware setup.

## Accessing the machine

The LinuxCNC controller runs on a **Raspberry Pi 4** with a static LAN IP address.

| | |
|---|---|
| **IP address** | `192.168.0.10` |
| **Username** | `cnc` |
| **Password** | `cnc` |

### Local login

Log in directly on the Raspberry Pi using the credentials above.

### Remote access via SSH

From any computer on the same network:

```bash
ssh cnc@192.168.0.10
```

Enter the password `cnc` when prompted.

Useful remote tasks over SSH:

```bash
# Start LinuxCNC headlessly (remote shell display mode)
linuxcnc axis_mm_minas_a6bf.ini

# Check EtherCAT slave status
ethercat slaves

# Check EtherCAT slave status with vendor/product IDs
ethercat slaves -v

# Inspect real-time HAL signals while LinuxCNC is running
halcmd show pin cia402.0
```

> :bulb: **NOTE:**
> LinuxCNC requires a real-time kernel. Graphical interfaces (AXIS, Gmoccapy, etc.) need a local display or X11 forwarding (`ssh -X cnc@192.168.0.10`). The headless `linuxcncrsh` display mode set in this configuration allows control via remote commands without a display.

---

## Prerequisites

The minimal prerequisites, in terms of configuration files, are

- [ethercat-conf-minas-a6bf.xml](config/ethercat-conf-minas-a6bf.xml)
  It is responsible for the protocol of EtherCAT communication containing SDO and PDO index.
- [cia402-minas-a6bf.hal](config/cia402-minas-a6bf.hal)
  `.hal` file is a script to set up a **hardware abstraction layer** (**HAL**) between LinuxCNC and your hardware, such that we can place instructions such as what modules have to be mounted and the relationship between them.
- [axis_mm_minas_a6bf.ini](config/axis_mm_minas_a6bf.ini)
  `.ini` file is the primary configuration passed to the command `linuxcnc` as a parameter to bring up LinuxCNC application. It is also where the settings relative to the three axes and joints are stored at. For instance, we define the limits of how far each axis can move and how fast each motor can rotate in this file. In a nutshell, it declares the final output of your 3-axial platform.

After editing these files, they should be placed under `~/linuxcnc/config/ethercat` in the device where LinuxCNC is living on.

> :bulb: **NOTE:**
> ​The names of configuration files are free to be named as you like, but remember to ensure if the dependency between these configuration files is correct. The dependency comes in by one file references other files.

---

## Documentation

The `doc/` folder contains reference guides that explain every variable in the configuration files and how to adapt them for different hardware.

| Document | Description |
|----------|-------------|
| [doc/ini-reference.md](doc/ini-reference.md) | Every section and variable in the INI file — machine identity, display settings, motion limits, kinematics, homing parameters, and scale values — with descriptions and adjustment notes. |
| [doc/hal-reference.md](doc/hal-reference.md) | Every command in the HAL file — module loading, servo-thread execution order, `setp` parameters, and the signal chains that wire the `cia402` component to the EtherCAT driver and the LinuxCNC motion controller. Includes a full `cia402` HAL pin reference table. |
| [doc/ethercat-xml-reference.md](doc/ethercat-xml-reference.md) | Every XML element and attribute in the EtherCAT configuration — master timing, slave identification (`vid`/`pid`), distributed clock setup, boot-time SDO configuration, and the complete PDO mapping with CiA 402 object dictionary. |
| [doc/scale-calculation-guide.md](doc/scale-calculation-guide.md) | Step-by-step derivation of `pos-scale`, `SCALE`/`INPUT_SCALE`, velocity/acceleration limits, and following-error limits from motor encoder resolution and mechanical drive ratio. Includes a worked example for the MINAS-A6BF. |
| [doc/linuxcncrsh-quickstart.md](doc/linuxcncrsh-quickstart.md) | Step-by-step guide for connecting to LinuxCNC over the network via `telnet` and the `linuxcncrsh` text interface — session handshake, E-stop, homing, MDI commands, program execution, and a concise command reference. |

### Official LinuxCNC documentation

The `doc/linuxcnc/` folder contains the official reference PDFs downloaded from [linuxcnc.org](https://linuxcnc.org/):

- `doc/linuxcnc/LinuxCNC_Documentation.pdf` — full LinuxCNC user and reference manual.
- `doc/linuxcnc/LinuxCNC_Developer.pdf` — HAL component development and internals.
- `doc/linuxcnc/LinuxCNC_Manual_Pages.pdf` — man pages for all LinuxCNC commands, including the full `linuxcncrsh` command reference.

---

## Quick start checklist

Use this checklist as a starting point when adapting the configuration to new hardware:

1. **Identify your drive** — run `ethercat slaves -v` on the target machine and note the `Vendor ID` and `Product code`. Update `vid` and `pid` in the EtherCAT XML file accordingly.
2. **Set `pos-scale`** — calculate encoder counts per mm from your encoder resolution and ball-screw pitch (see [scale-calculation-guide.md](doc/scale-calculation-guide.md)). Apply with `setp cia402.N.pos-scale <value>` in the HAL file.
3. **Set joint and axis limits** — measure the actual travel range of each axis. Set `MIN_LIMIT` / `MAX_LIMIT` in `[AXIS_N]` and `[JOINT_N]` a few mm inside the hard-stop positions.
4. **Set velocity and acceleration** — derive `MAX_VELOCITY` (mm/s) from the motor's rated speed and mechanical drive ratio. Set it in both `[AXIS_N]` and `[JOINT_N]`. Start with a conservative `MAX_ACCELERATION`.
5. **Match servo loop timing** — confirm that `appTimePeriod` in the EtherCAT XML equals `SERVO_PERIOD` in the INI file (both in nanoseconds).
6. **Verify** — jog each axis 100 mm and measure the actual displacement with a dial gauge. Commanded distance and actual distance should match within encoder resolution.
