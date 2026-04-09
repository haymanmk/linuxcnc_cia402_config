# EtherCAT XML Reference

This document explains every XML element and attribute in [`config/ethercat-conf-minas-a6bf.xml`](../config/ethercat-conf-minas-a6bf.xml).

The EtherCAT XML configuration file tells `linuxcnc-ethercat` (`lcec`) how the EtherCAT bus is structured: which masters are active, which servo drives are connected, and how each drive's PDO (Process Data Object) variables map to LinuxCNC HAL pins.

This file is loaded at LinuxCNC start-up by the following HAL command:

```hal
loadusr -W lcec_conf ethercat-conf-minas-a6bf.xml
```

---

## Document structure

```xml
<masters>
  <master idx="0" appTimePeriod="1000000" refClockSyncCycles="1">

    <slave idx="0" type="generic" vid="0000066F" pid="613C0006" configPdos="true">

      <!-- Boot-time SDO configuration (optional) -->
      <sdoConfig idx="6098" subIdx="00">
        <sdoDataRaw data="23"/>
      </sdoConfig>

      <!-- Distributed clock settings -->
      <dcConf assignActivate="300" sync0Cycle="*1" sync0Shift="0"/>

      <!-- Output PDO (master → drive): RxPDO -->
      <syncManager idx="2" dir="out">
        <pdo idx="1600">
          <pdoEntry idx="6040" subIdx="00" bitLen="16" halPin="cia-controlword" halType="u32"/>
          ...
        </pdo>
      </syncManager>

      <!-- Input PDO (drive → master): TxPDO -->
      <syncManager idx="3" dir="in">
        <pdo idx="1a00">
          <pdoEntry idx="6041" subIdx="00" bitLen="16" halPin="cia-statusword" halType="u32"/>
          ...
        </pdo>
      </syncManager>

    </slave>

    <!-- Additional slaves for Y and Z axes (identical structure, idx="1" and idx="2") -->

  </master>
</masters>
```

---

## `<masters>`

The root element of the file. Contains one or more `<master>` elements. Has no attributes.

---

## `<master>` — EtherCAT master configuration

Describes one EtherCAT bus (one network adapter running the EtherCAT master stack).

| Attribute | Example | Required | Description |
|-----------|---------|----------|-------------|
| `idx` | `0` | Optional (default: 0) | Index of the EtherCAT master interface. Systems with a single EtherCAT NIC use `idx="0"`. Verify with `ethercat slaves` — the output header shows `Master0`, `Master1`, etc. |
| `appTimePeriod` | `1000000` | Required | EtherCAT cycle time in **nanoseconds**. **Must equal `[EMCMOT] SERVO_PERIOD` in the INI file.** `1000000` = 1 ms (1 kHz). If these values differ, LinuxCNC will report a synchronisation warning. |
| `refClockSyncCycles` | `1` | Required | How often to resynchronise the distributed clocks across all slaves, expressed as a multiple of `appTimePeriod`. `1` syncs every cycle (most accurate and typical for motion control). `1000` would sync every second. |
| `syncToRefClock` | *(not set in example)* | Optional | Set to `"true"` to synchronise the LinuxCNC servo thread itself to the EtherCAT distributed clock reference. Recommended when using DC synchronisation (`<dcConf>`) for tighter jitter control. |

**Adjusting for your hardware:**
- Change `appTimePeriod` only together with `SERVO_PERIOD` in the INI (see [`scale-calculation-guide.md`](scale-calculation-guide.md)).
- Add `syncToRefClock="true"` for multi-slave systems that use DC synchronisation.

---

## `<slave>` — EtherCAT slave (servo drive) configuration

One `<slave>` block per servo drive. This example has three identical blocks for indices `0`, `1`, and `2` (X, Y, Z axes).

| Attribute | Example | Required | Description |
|-----------|---------|----------|-------------|
| `idx` | `0`, `1`, `2` | Required | Slave position on the EtherCAT bus, **zero-based**, in physical daisy-chain order. The drive plugged directly into the master NIC is `idx="0"`, the next in the chain is `idx="1"`, and so on. Verify order with `ethercat slaves`. |
| `type` | `generic` | Required | Driver type. `"generic"` means all PDO mapping is defined entirely in this XML file — no compiled driver code is needed. For devices with compiled drivers in `linuxcnc-ethercat`, you can use the device type name (e.g., `"EL7041"`) instead. |
| `vid` | `0000066F` | Required for `generic` | **Vendor ID** (hexadecimal). Identifies the manufacturer. `0000066F` is Panasonic. Determine your device's VID by running `ethercat slaves -v`. |
| `pid` | `613C0006` | Required for `generic` | **Product ID** (hexadecimal). Identifies the drive model. `613C0006` corresponds to the MINAS-A6BF series. Also from `ethercat slaves -v`. |
| `configPdos` | `true` | Required for `generic` | Instructs `lcec` to apply the PDO mapping defined in this XML to the slave. Always set to `"true"` when using the generic driver with custom PDOs. |
| `name` | *(not set)* | Optional | Human-readable label for the slave. If set, HAL pin names become `lcec.<master>.<name>.*` instead of `lcec.<master>.<idx>.*`. Example: `name="x-drive"` would produce pins like `lcec.0.x-drive.actual-position`. |

**Adjusting for your hardware:**
- Run `ethercat slaves -v` to identify `vid` and `pid` for each drive. The output looks like:  
  `0  0:0  PREOP  +  MBDLT25BF  Vendor ID: 0x0000066F, Product code: 0x613C0006`
- When adding a new axis, copy an existing `<slave>` block and increment `idx`.

---

## `<sdoConfig>` and `<sdoDataRaw>` — Boot-time SDO configuration

SDOs (Service Data Objects) allow writing to a drive's object dictionary during EtherCAT initialisation, before the drive enters operational mode. Use this to set drive parameters that would otherwise require manual configuration through drive software.

The example block is commented out but illustrates how to set the homing method:

```xml
<!-- Uncomment to configure the homing method during bootup. -->
<sdoConfig idx="6098" subIdx="00">
  <sdoDataRaw data="23"/>
</sdoConfig>
```

| Attribute / Element | Example | Description |
|--------------------|---------|-------------|
| `<sdoConfig idx="...">` | `idx="6098"` | CiA 402 object index to write (hexadecimal). `0x6098` is the **Homing Method** object. |
| `subIdx="..."` | `subIdx="00"` | Sub-index (hexadecimal). `00` is the primary sub-index for single-value objects. |
| `<sdoDataRaw data="..."/>` | `data="23"` | Value to write, in **hexadecimal**. `0x23` = decimal 35 = CiA 402 homing method 35. |

### Common CiA 402 homing methods (object 0x6098)

| Decimal | Hex | Description |
|---------|-----|-------------|
| 1 | `01` | Search for home on negative limit switch |
| 2 | `02` | Search for home on positive limit switch |
| 17 | `11` | Home on negative limit switch + index pulse |
| 18 | `12` | Home on positive limit switch + index pulse |
| 33 | `21` | Home on negative home switch |
| 34 | `22` | Home on positive home switch |
| 35 | `23` | Set current position as home without any movement |
| -1 | `FF` | Drive-specific homing method |

**Adjusting for your hardware:**
- If your drive needs a specific homing method configured at boot, uncomment this block and set `data` to the appropriate hex value.
- You can add multiple `<sdoConfig>` blocks per slave (e.g., to set homing speed, acceleration, or current limits).
- Consult your drive's manual for supported object indices and valid values.

---

## `<dcConf>` — Distributed Clock configuration

Enables EtherCAT Distributed Clocks (DC) synchronisation for a slave. DC synchronisation ensures all slaves in the network latch their I/O data at precisely the same instant, reducing position jitter in multi-axis systems.

```xml
<dcConf assignActivate="300" sync0Cycle="*1" sync0Shift="0"/>
```

| Attribute | Example | Description |
|-----------|---------|-------------|
| `assignActivate` | `300` | DC activation bitmask in **hexadecimal**. `0x0300` is the standard value for most CiA 402 servo drives with DC support. Some drives require `0x0700`. Look up the correct value in the drive's ESI (EtherCAT Slave Information) XML file. |
| `sync0Cycle` | `*1` | Sync0 pulse period. `*1` means one multiple of `appTimePeriod` (i.e., every cycle = every 1 ms). `*2` would fire every other cycle. |
| `sync0Shift` | `0` | Phase offset in nanoseconds for the Sync0 pulse. `0` aligns it with the start of the cycle. A positive value delays the pulse (useful when computation takes a known fixed time). |

**Adjusting for your hardware:**
- For the MINAS-A6BF and most modern CiA 402 drives, `assignActivate="300"` with `sync0Cycle="*1"` and `sync0Shift="0"` is the correct starting point.
- If DC synchronisation causes issues (drive faults, jitter), check the `assignActivate` value in the drive's ESI file or manufacturer documentation.

---

## `<syncManager>` — PDO sync manager (RxPDO / TxPDO)

A sync manager defines a block of process data that is exchanged with the slave every cycle. Each slave in this configuration has two sync managers: one for output data (commands to drive) and one for input data (status from drive).

```xml
<syncManager idx="2" dir="out"> ... </syncManager>  <!-- Output to drive: RxPDO -->
<syncManager idx="3" dir="in">  ... </syncManager>  <!-- Input from drive: TxPDO -->
```

| Attribute | Example | Description |
|-----------|---------|-------------|
| `idx` | `2`, `3` | Sync manager index, defined by the drive hardware. For Panasonic MINAS-A6BF: SM2 = output (RxPDO), SM3 = input (TxPDO). These numbers are hardware-specific — check your drive's ESI file. |
| `dir` | `"out"`, `"in"` | Data direction from the EtherCAT master's perspective. `"out"` = master writes to slave (RxPDO). `"in"` = master reads from slave (TxPDO). |

> **Naming convention:** RxPDO and TxPDO are named from the *slave's* perspective. The slave *receives* (Rx) the data the master writes, and *transmits* (Tx) the data the master reads. Therefore `dir="out"` = RxPDO and `dir="in"` = TxPDO.

---

## `<pdo>` — PDO mapping index

Groups one or more `<pdoEntry>` elements under a single PDO (Process Data Object). A PDO is a fixed-size block of cyclic data.

```xml
<pdo idx="1600"> ... </pdo>
```

| Attribute | Example | Description |
|-----------|---------|-------------|
| `idx` | `1600`, `1a00` | PDO index in hexadecimal. Must match the drive's PDO mapping object indices: `0x1600–0x17FF` for RxPDO, `0x1A00–0x1BFF` for TxPDO. Verify with the drive's ESI file. |

The MINAS-A6BF default PDO assignments:
- RxPDO (output, SM2): index `0x1600`
- TxPDO (input, SM3): index `0x1A00`

---

## `<pdoEntry>` — Individual object entry within a PDO

Each `<pdoEntry>` maps one CiA 402 object dictionary entry to a LinuxCNC HAL pin.

```xml
<pdoEntry idx="6040" subIdx="00" bitLen="16" halPin="cia-controlword" halType="u32"/>
```

| Attribute | Example | Required | Description |
|-----------|---------|----------|-------------|
| `idx` | `6040` | Required | CiA 402 object index (hexadecimal). Objects in the `0x6000–0x7FFF` range are CiA 402 device profile objects. |
| `subIdx` | `00` | Required | Object sub-index (hexadecimal). `00` for single-value objects; higher numbers access sub-entries of record/array objects. |
| `bitLen` | `16` | Required | Bit width of the PDO entry. Must match the drive's ESI specification exactly. Common values: `8` (USINT/SINT), `16` (UINT/INT), `32` (UDINT/DINT). |
| `halPin` | `cia-controlword` | Optional | Name of the LinuxCNC HAL pin. The full pin name becomes `lcec.<master-idx>.<slave-idx>.<halPin>`. If `halPin` is omitted, the entry occupies space in the PDO but has no HAL pin (used for alignment/padding). |
| `halType` | `u32` | Optional | LinuxCNC HAL data type. `bit` (boolean), `s32` (signed 32-bit), `u32` (unsigned 32-bit), `float` (32-bit float mapped from integer), `complex` (sub-fields). Match to the object's semantic meaning. |

---

## CiA 402 object dictionary — objects used in this configuration

These object indices are standardised by the CiA 402 specification and are the same on all compliant drives.

| Object | Sub | Bit width | HAL pin | Type | Direction | Description |
|--------|-----|-----------|---------|------|-----------|-------------|
| `6040` | `00` | 16 | `cia-controlword` | `u32` | RxPDO (out) | **Controlword** — commands the drive state machine. Individual bits trigger transitions: enable, quick stop, fault reset, operation-specific commands. |
| `6041` | `00` | 16 | `cia-statusword` | `u32` | TxPDO (in) | **Statusword** — reports current drive state. Decoded by `cia402` into individual `stat-*` pins. |
| `6060` | `00` | 8 | `opmode` | `s32` | RxPDO (out) | **Modes of Operation** — selects drive operating mode: `6` = homing, `8` = CSP (Cyclic Synchronous Position), `9` = CSV (Cyclic Synchronous Velocity). |
| `6061` | `00` | 8 | `opmode-display` | `s32` | TxPDO (in) | **Modes of Operation Display** — reads back the drive's currently active mode. |
| `607A` | `00` | 32 | `target-position` | `s32` | RxPDO (out) | **Target Position** — commanded absolute position in encoder counts. Used in CSP mode. |
| `60FF` | `00` | 32 | `target-velocity` | `s32` | RxPDO (out) | **Target Velocity** — commanded velocity in drive-internal units. Used in CSV mode. |
| `6064` | `00` | 32 | `actual-position` | `s32` | TxPDO (in) | **Position Actual Value** — current encoder position in drive counts. The number of counts per revolution depends on drive configuration (electronic gearing). |
| `606C` | `00` | 32 | `actual-velocity` | `s32` | TxPDO (in) | **Velocity Actual Value** — current motor velocity in drive-internal units. |
| `6077` | `00` | 16 | `actual-torque` | `s32` | TxPDO (in) | **Torque Actual Value** — current motor torque. Unit: per-mil (‰) of rated torque. `1000` = 100% rated torque. |
| `6098` | `00` | 8 | *(SDO only)* | — | SDO write | **Homing Method** — configured via `<sdoConfig>` during boot; not typically in the cyclic PDO. |

---

## Adding a new drive (fourth axis)

1. Copy an existing `<slave>` block and set `idx="3"`. Verify `vid` and `pid` match the new drive via `ethercat slaves -v`.
2. In the HAL file: increase `count=` in `loadrt cia402 count=4`, add the `addf` and `net` lines for the new instance (`cia402.3.*`), and connect it to `lcec.0.3.*`.
3. In the INI file: add `[JOINT_3]` and `[AXIS_W]` sections, update `[KINS] JOINTS = 4`, and add `W` to `[TRAJ] COORDINATES = X Y Z W`.

---

## Cross-references

- [ini-reference.md](ini-reference.md) — `appTimePeriod` must equal `SERVO_PERIOD`.
- [hal-reference.md](hal-reference.md) — how `lcec.*` HAL pins are used in `net` commands.
- [scale-calculation-guide.md](scale-calculation-guide.md) — how object 0x6064 encoder resolution affects `pos-scale`.
