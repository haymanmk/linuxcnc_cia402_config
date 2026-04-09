# linuxcncrsh Quickstart — Controlling LinuxCNC via Telnet

`linuxcncrsh` is a text-mode display program built into LinuxCNC. Instead of opening a graphical window, it listens on a TCP port and accepts commands from any telnet or netcat client on the network. This configuration already uses it as the display (`DISPLAY = linuxcncrsh` in the INI file), so no additional setup is needed on the machine.

> :warning: **Security note:** `linuxcncrsh` passwords (connect and enable) are transmitted in **plain text**. Only use this interface on a trusted private network. Never expose port 5007 to the internet.

---

## Connection details

| Parameter | Value |
|-----------|-------|
| Host | `192.168.0.10` (Raspberry Pi LAN IP) |
| Port | `5007` (default) |
| Connect password | `EMC` (default) |
| Enable password | `EMCTOO` (default) |

---

## Connecting

From any computer on the same network:

```bash
telnet 192.168.0.10 5007
```

You should see:

```
Trying 192.168.0.10...
Connected to 192.168.0.10
Escape character is '^]'.
```

LinuxCNC is now waiting for your first command.

---

## Session walkthrough

The commands below are typed by you. Responses from the server are shown indented.

### 1. Handshake

```
> hello EMC my-client 1.0
HELLO ACK EMCNETSVR 1.1
```

`hello <connect-password> <client-name> <protocol-version>`
- `EMC` — default connect password.
- `my-client` — any identifier for your session (no spaces).
- `1.0` — protocol version; `1.0` or `1.1` are accepted.

### 2. Enable machine control

```
> set enable EMCTOO
set enable EMCTOO
```

The server echoes the command back as acknowledgement. Until `enable` is set with the correct password, `set` commands are rejected. `get` (read-only) commands work without it.

### 3. Take the machine out of E-stop

```
> set estop off
set estop off
```

### 4. Turn the machine on

```
> set machine on
set machine on
```

### 5. Home all axes (Z first, then X and Y)

Because `HOME_SEQUENCE` is set to home Z first, you can home all joints with a single command:

```
set home -1
```

Or home individual joints one by one:

```
set home 2
set home 0
set home 1
```

`home <joint>` — joint numbers: `0` = X, `1` = Y, `2` = Z. `-1` = home all.

### 6. Switch to MDI mode and run a G-code command

```
set mode mdi
set mdi g0 x10 y10 z5
```

### 7. Check actual position

```
> get abs_act_pos
abs_act_pos 10.000000 10.000000 5.000000
```

### 8. Run a G-code file

```
set mode auto
set open /home/cnc/linuxcnc/nc_files/my-program.ngc
set run
```

Check status while running:

```
get program_status
get program_line
```

### 9. Disconnect and shut down LinuxCNC

```
> shutdown
shutdown
Connection closed by foreign host.
```

---

## Essential commands reference

### Session

| Command | Description |
|---------|-------------|
| `hello <pw> <client> <version>` | Initiate handshake. Required before any other command. |
| `set enable <enablepw>` | Unlock control commands for this session. |
| `set enable off` | Lock control commands (read-only mode). |
| `quit` | Disconnect without shutting down LinuxCNC. |
| `shutdown` | Disconnect and shut down LinuxCNC (requires enable). |
| `help` | List available commands. |
| `help <command>` | Show usage for a specific command. |

### Machine state

| Command | Description |
|---------|-------------|
| `set estop off` | Clear E-stop (machine ready to enable). |
| `set estop on` | Trigger E-stop. |
| `set machine on` | Turn machine on (servo drives energised). |
| `set machine off` | Turn machine off. |
| `set mode manual` | Manual mode (jogging). |
| `set mode mdi` | MDI mode (single G-code commands). |
| `set mode auto` | Auto mode (run a G-code file). |
| `get estop` | Returns `on` or `off`. |
| `get machine` | Returns `on` or `off`. |
| `get mode` | Returns `manual`, `mdi`, or `auto`. |
| `get error` | Returns current error string, or `ok`. |

### Homing

| Command | Description |
|---------|-------------|
| `set home -1` | Home all joints (respects `HOME_SEQUENCE` order). |
| `set home <n>` | Home joint `n` (0 = X, 1 = Y, 2 = Z). |
| `get joint_homed` | Returns homed status for all joints. |
| `get joint_homed <n>` | Returns `homed` or `not` for joint `n`. |

### Position

| Command | Description |
|---------|-------------|
| `get abs_act_pos` | Actual positions of all axes (absolute machine coordinates). |
| `get abs_act_pos <n>` | Actual position of axis `n`. |
| `get rel_act_pos` | Actual positions in current work coordinate system. |
| `get joint_pos` | Raw joint positions (no tool offset). |
| `get joint_limit` | Limit status for all joints (`ok`, `minsoft`, `maxsoft`, `minhard`, `maxhard`). |
| `get joint_fault` | Fault status for all joints (`ok` or `fault`). |

### Jogging (manual mode only)

| Command | Description |
|---------|-------------|
| `set jog <joint> <speed>` | Start continuous jog on joint at speed (mm/s); sign = direction. |
| `set jog_incr <joint> <speed> <dist>` | Jog joint by `dist` mm at `speed`. |
| `set jog_stop <joint>` | Stop jog on joint. |

### MDI and programs

| Command | Description |
|---------|-------------|
| `set mdi <gcode>` | Run a single G-code line (requires MDI mode). |
| `set open <filepath>` | Open a G-code file (requires auto mode). |
| `set run` | Start program from beginning. |
| `set run <line>` | Start program from line number. |
| `set pause` | Pause program execution. |
| `set resume` | Resume paused program. |
| `set abort` | Abort program or MDI execution. |
| `set step` | Step one program line. |
| `get program_status` | Returns `idle`, `running`, or `paused`. |
| `get program_line` | Returns the currently executing line number. |
| `get program` | Returns the name of the open program. |

### Feed and spindle overrides

| Command | Description |
|---------|-------------|
| `set feed_override <pct>` | Set feed override in percent (e.g., `80` = 80%). |
| `get feed_override` | Get current feed override. |
| `set spindle_override <pct>` | Set spindle speed override in percent. |

---

## Verbose mode

Enable verbose mode to receive explicit `ACK` confirmations for every `set` command:

```
set verbose on
```

Disable with `set verbose off`. Verbose defaults to `off` on new connections.

---

## Useful one-liners from the shell

These use `nc` (netcat) to send a batch of commands non-interactively:

```bash
# Check whether LinuxCNC has faulted on any joint
echo -e "hello EMC check 1.0\nget joint_fault\nquit" | nc 192.168.0.10 5007

# Read all actual positions
echo -e "hello EMC check 1.0\nget abs_act_pos\nquit" | nc 192.168.0.10 5007

# Emergency stop
echo -e "hello EMC ctrl 1.0\nset enable EMCTOO\nset estop on\nquit" | nc 192.168.0.10 5007
```

---

## Full command reference

See `doc/linuxcnc/LinuxCNC_Manual_Pages.pdf` in this repository for the complete `linuxcncrsh` man page, including all subcommands and their exact syntax.
