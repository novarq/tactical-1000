# Tactical 1000 Boot Modes

## Overview

The Tactical 1000 uses the LAN9696 `VCORE[3:0]` bootstrap pins to select its boot mode. The MCP2200 on the USB-C console controls these pins and the board reset line.

The LAN9696 samples the straps during power-on reset and pin reset. To change mode:

1. Set `VCORE[3:0]` while reset is asserted.
2. Wait for the signals to settle.
3. Release reset.

Changing the straps without resetting the LAN9696 does not change the active mode.

## MCP2200 Pin Mapping

| Bit | MCP2200 pin | Tactical 1000 use |
|---:|---|---|
| 7 | `GP7` | LED connection; not part of boot selection |
| 6 | `GP6` | LED connection; not part of boot selection |
| 5 | `GP5` | `VCORE3` |
| 4 | `GP4` | `VCORE2` |
| 3 | `GP3` | `VCORE1` |
| 2 | `GP2` | `VCORE0` |
| 1 | `GP1` | Unused input |
| 0 | `GP0` | Reset: `0` asserted, `1` released |

`GP1` must remain configured as an input. Its output-latch value does not drive the physical pin. `GP7:6` do not select the boot mode.

## Complete Boot-Mode Table

| `VCORE[3:0]` | Tool mode | Meaning |
|---|---|---|
| `0000` | `emmc-trace` | Boot from eMMC with boot trace on FLEXCOM0 |
| `0001` | `qspi0-trace` | Boot from QSPI0 with boot trace on FLEXCOM0 |
| `0010` | `sdcard-trace` | Boot from SD Card with boot trace on FLEXCOM0 |
| `0011` | `emmc` | Boot from eMMC without ROM boot trace |
| `0100` | `qspi0` | Boot from QSPI0 without ROM boot trace |
| `0101` | `sdcard` | Boot from SD Card without ROM boot trace |
| `0110` | Reserved | Do not use |
| `0111` | Reserved | Do not use |
| `1000` | `qspi0-hs-trace` | TF-A monitor on FLEXCOM0 with OTP emulation loaded from QSPI0 |
| `1001` | Reserved | Do not use |
| `1010` | `tfa-monitor` | Run the firmware-update monitor at 115200 baud |
| `1011` | `tfa-monitor-hs` | Run the firmware-update monitor at 921600 baud |
| `1100` | Reserved | Do not use |
| `1101` | `qspi0-hs` | Boot from the high-speed QSPI0 interface |
| `1110` | Reserved | Do not use |
| `1111` | `spi-client` | Disable automatic internal CPU boot and use QSPI0/SI as an SPI client |

Reserved modes must not be used. The CLI currently names `1000` as `qspi0-hs-trace`; the LAN969x data sheet describes that strap as the TF-A monitor on FC0 with OTP emulation loaded from QSPI0.

## Raw Boot Values

The boot-control bits are:

```text
asserted = VCORE << 2
released = asserted | 0x01
```

The following values match a configuration such as the observed `0x01` eMMC default. Bits 7, 6, and 1 do not select the LAN9696 boot mode.

| Mode | `VCORE` | Reset asserted | Reset released | UART |
|---|---|---:|---:|---:|
| eMMC with trace | `0000` | `0x00` | `0x01` | 115200 |
| QSPI0 with trace | `0001` | `0x04` | `0x05` | 115200 |
| eMMC without ROM trace | `0011` | `0x0c` | `0x0d` | Set by later firmware |
| QSPI0 without ROM trace | `0100` | `0x10` | `0x11` | Set by later firmware |
| FC0 monitor | `1010` | `0x28` | `0x29` | 115200 |
| FC0-HS monitor | `1011` | `0x2c` | `0x2d` | 921600 |

Full MCP2200 values can differ in non-boot bits while selecting the same mode. For example, all these pairs select `VCORE=1011`:

| Reset asserted | Reset released |
|---:|---:|
| `0x2c` | `0x2d` |
| `0xec` | `0xed` |
| `0xee` | `0xef` |

Decode any full byte with:

```text
VCORE          = (value >> 2) & 0x0f
reset_released = value & 0x01
```

Use the named tool commands below instead of writing raw full-byte values.

## UART Rates

Boot mode selects the LAN9696 ROM behavior, but it does not configure the host serial program.

| ROM behavior | Host serial setting |
|---|---|
| FC0 boot trace | 115200, 8-N-1 |
| FC0 monitor (`1010`) | 115200, 8-N-1 |
| FC0-HS monitor (`1011`) | 921600, 8-N-1 |
| No-trace storage mode | No ROM trace; later firmware selects its console rate |

The TF-A monitor is a firmware-transfer service, not a storage source. Entering monitor mode does not itself write eMMC or QSPI0.

## Using `mcp2200ctl.py`

The Tactical 1000-aware Python tool is available from:

https://github.com/novarq/mcp2200py

Clone it with:

```sh
git clone https://github.com/novarq/mcp2200py.git
cd mcp2200py
```

Install the declared Python dependency:

```sh
python3 -m pip install -r requirements.txt
```

The repository requires `hidapi>=0.14`. The account running the tool must have read/write permission on the MCP2200 `/dev/hidrawN` node.

List matching devices:

```sh
python3 mcp2200ctl.py list
```

Read the persistent configuration and current GPIO state:

```sh
python3 mcp2200ctl.py get-config --path /dev/hidrawN
python3 mcp2200ctl.py get-gpio --path /dev/hidrawN
```

Replace `/dev/hidrawN` with the device reported on the host.

### Configure GP1 as input

Set `GP1` as input and keep the other GPIO direction bits clear:

```sh
python3 mcp2200ctl.py set-config \
    --path /dev/hidrawN --io 0x02 --verify
```

The verified configuration should report:

```text
Directions: 0x02 (0b00000010)
```

The complete Tactical 1000 initializer also configures GP1, reset, LED roles, UART defaults, and related options:

```sh
python3 mcp2200ctl.py init-novarq-tactical-1000 \
    --path /dev/hidrawN
```

This command displays the complete profile and asks for confirmation before writing persistent configuration.

## Select a Temporary Boot Mode

Use the semantic boot-mode command. It changes only the live `GP5:2` strap outputs and verifies them.

For the high-speed TF-A monitor:

```sh
python3 mcp2200ctl.py set-tactical-1000-boot-mode \
    tfa-monitor-hs --path /dev/hidrawN
python3 mcp2200ctl.py reset-tactical-1000 \
    --path /dev/hidrawN
```

Then open the serial port at 921600 baud:

```sh
picocom -b 921600 /dev/ttyACM0
```

Other common selections are:

```sh
# eMMC with FC0 trace
python3 mcp2200ctl.py set-tactical-1000-boot-mode \
    emmc-trace --path /dev/hidrawN

# eMMC without ROM trace
python3 mcp2200ctl.py set-tactical-1000-boot-mode \
    emmc --path /dev/hidrawN

# QSPI0 with FC0 trace
python3 mcp2200ctl.py set-tactical-1000-boot-mode \
    qspi0-trace --path /dev/hidrawN

# QSPI0 without ROM trace
python3 mcp2200ctl.py set-tactical-1000-boot-mode \
    qspi0 --path /dev/hidrawN

# TF-A monitor at 115200 baud
python3 mcp2200ctl.py set-tactical-1000-boot-mode \
    tfa-monitor --path /dev/hidrawN
```

After selecting one mode, run `reset-tactical-1000` so the LAN9696 samples it. Both commands ask for confirmation unless `--yes` is supplied.

## Persistent eMMC and QSPI0 Boot

The MCP2200 persistent GPIO default determines the strap value restored when the MCP2200 resets or powers up.

| Persistent boot source | ROM trace | Default value |
|---|---|---:|
| eMMC | Enabled on FC0 | `0x01` |
| QSPI0 | Enabled on FC0 | `0x05` |
| eMMC | Disabled | `0x0d` |
| QSPI0 | Disabled | `0x11` |

`set-config` preserves omitted configuration fields. Always use `--verify` when changing the persistent default.

### Persist eMMC boot

Persist eMMC boot with FC0 trace:

```sh
python3 mcp2200ctl.py set-config --path /dev/hidrawN \
    --default-values 0x01 --verify
```

Persist eMMC boot without ROM trace:

```sh
python3 mcp2200ctl.py set-config --path /dev/hidrawN \
    --default-values 0x0d --verify
```

To apply the selection immediately, set the matching live mode and reset the LAN9696:

```sh
python3 mcp2200ctl.py set-tactical-1000-boot-mode \
    emmc-trace --path /dev/hidrawN
python3 mcp2200ctl.py reset-tactical-1000 \
    --path /dev/hidrawN
```

### Persist QSPI0 boot

Persist QSPI0 boot with FC0 trace:

```sh
python3 mcp2200ctl.py set-config --path /dev/hidrawN \
    --default-values 0x05 --verify
```

Persist QSPI0 boot without ROM trace:

```sh
python3 mcp2200ctl.py set-config --path /dev/hidrawN \
    --default-values 0x11 --verify
```

To apply the trace-suppressed selection immediately:

```sh
python3 mcp2200ctl.py set-tactical-1000-boot-mode \
    qspi0 --path /dev/hidrawN
python3 mcp2200ctl.py reset-tactical-1000 \
    --path /dev/hidrawN
```

Changing the MCP2200 default UART baud does not select a LAN9696 boot mode or configure the active host terminal.

## Troubleshooting

### Permission denied opening `/dev/hidrawN`

Run an initial check with `sudo`, or install a udev rule granting the operator access to MCP2200 HID devices.

### Serial output is silent or garbled

Check the selected mode and terminal baud rate. FC0 uses 115200; FC0-HS uses 921600.

### A mode change is rejected

Run `get-config`. `GP2` through `GP5` must be GPIO outputs. Apply `init-novarq-tactical-1000` if the MCP2200 does not have the board profile.

### Reset is rejected

Run `get-config`. `GP0` must be a normal GPIO output rather than an input or alternate-function pin.

## References

- Microchip LAN969x data sheet, Section 2.2.1, "Configuration Straps."
- Microchip LAN969x data sheet, Section 2.2.2, UART-rate note.
- Microchip LAN969x data sheet, Section 3.47.2, "VCore Configurations."
- [`mcp2200ctl.py`](https://github.com/novarq/mcp2200py).
- [Serial Console Access Guide](serial-console-access.md).
- [OpenWrt Installation and Configuration Guide](openwrt-setup-guide.md).
