# OpenWrt Installation and Configuration Guide

## Introduction

This guide walks you through installing OpenWrt on the Novarq Tactical 1000 and getting it ready for everyday use. The Tactical 1000 is supported by mainline OpenWrt as a `microchipsw/lan969x` target board (`novarq_tactical-1000`), built around the Microchip LAN9696 switch SoC with hardware switching exposed through the Linux switchdev framework.

There are two paths covered:

1. **Initial installation** - flashing a full eMMC image over the built-in USB-C console using the LAN969x Firmware Update (FWU) tool. This is the path you take the first time, or whenever you need to recover the device.
2. **Routine upgrades** - once OpenWrt is running, regular `sysupgrade` is the supported way to move to a newer build.

Before you begin, you should already be familiar with the serial console - see the [Serial Console Access Guide](https://github.com/novarq/tactical-1000/blob/main/docs/serial-console-access.md).

## OpenWrt Images

Pre-built snapshot images for the Tactical 1000 are published at:

https://downloads.openwrt.org/snapshots/targets/microchipsw/lan969x/

Two image artifacts are relevant for this board:

- **`openwrt-microchipsw-lan969x-novarq_tactical-1000-squashfs-emmc-gpt.img.gz`** - the full eMMC image. It contains the complete GPT partition layout: the primary and backup FIP (firmware) partitions, the U-Boot environment, and the kernel and root filesystem partitions. This is the image you install in bootstrap mode through `fwu.html`. Flashing it lays down a known-good factory layout in a single shot and is also the right choice for unbricking a device.
- **`openwrt-microchipsw-lan969x-novarq_tactical-1000-squashfs-sysupgrade.bin`** - the standard OpenWrt sysupgrade image. It carries only the kernel and rootfs payload and is intended for use with `sysupgrade` from a running OpenWrt system. It does not touch the FIP, environment, or partition table.

A rough rule of thumb: use `emmc-gpt.img.gz` to put OpenWrt on the device for the first time (or to recover); use `sysupgrade.bin` for everything after that.

## First-Time Installation via Bootstrap Mode

The Tactical 1000's USB-C console is driven by a Microchip MCP2200 USB-to-serial bridge. In addition to forwarding the UART, the MCP2200 exposes eight GPIOs (`GP0`-`GP7`) that are wired to the LAN9696 reset and VCORE boot-strap pins. The `mcpconfig` tool reads and writes those GPIOs as a single 8-bit byte. By writing the right value we hold the SoC in reset, select a boot mode on the VCORE straps, then release reset so the SoC comes up in **bootstrap mode** - where it accepts a firmware image from the host over the same USB-C cable using the web-based `fwu.html` flashing tool.

### Understanding the GPIO Byte

Each bit in the byte controls one MCP2200 pin:

| Bit | Pin   | Function on Tactical 1000        |
|-----|-------|----------------------------------|
| 0   | `GP0` | `nMR` board reset, active low (`1` = reset released) |
| 1   | `GP1` | LAN9696 GPIO63                   |
| 2   | `GP2` | VCORE0 boot strap                |
| 3   | `GP3` | VCORE1 boot strap                |
| 4   | `GP4` | VCORE2 boot strap                |
| 5   | `GP5` | VCORE3 boot strap                |
| 6   | `GP6` | unused for boot selection        |
| 7   | `GP7` | unused for boot selection        |

The boot mode is selected by the four VCORE strap bits `GP5..GP2`. `GP0` must be high after the mode is selected so reset is released. `GP6` and `GP7` do not affect boot - their state can vary between boards and either value is acceptable.

The values used below follow this pattern: first write the byte with `GP0=0` (reset held low) so the new straps latch cleanly, then write the same byte with `GP0=1` (reset released) so the SoC boots in the selected mode.

| Mode                            | VCORE `GP5..GP2` | UART baud | Reset low | Reset released     |
|---------------------------------|------------------|-----------|-----------|--------------------|
| eMMC FC0 (normal boot)          | `0000`           | 115200    | `0xc0`    | `0xc1` (or `0x01`) |
| QSPI FC0                        | `0001`           | 115200    | `0xc4`    | `0xc5`             |
| Bootstrap monitor / TF-A FC0    | `1010`           | 115200    | `0xe8`    | `0xe9`             |
| Bootstrap monitor / TF-A FC0-HS | `1011`           | 921600    | `0xee`    | `0xef`             |

The two bootstrap variants differ only in the UART baud rate of the ROM monitor: **FC0** runs at the default 115200 baud, while **FC0-HS** runs at 921600 baud, which makes flashing the multi-hundred-megabyte eMMC image several times faster. The procedure below uses FC0-HS.

### Prerequisites

- A Linux host with a USB-A to USB-C cable connected to the switch's console port.
- Google Chrome (or another Chromium-based browser that supports the Web Serial API) - `fwu.html` relies on it.
- The `mcpconfig` utility for driving the MCP2200 GPIOs.
- The `emmc-gpt.img.gz` image from the downloads link above, saved locally.

### Build the mcpconfig Tool

```sh
git clone https://github.com/ipaton0/mcp2200.git
cd mcp2200
make
```

This produces the `mcpconfig` binary used in the steps below.

### HID Access for IO Configuration

On modern Linux hosts the `hid_mcp2200` kernel module claims the MCP2200's HID interface by default. While the module is bound, the MCP2200 device will not be visible to the `mcpconfig` tool and no `/dev/usb/hiddevX` node is created for it. Unload the module so the HID interface is available to userspace:

```sh
sudo rmmod hid_mcp2200
```

The serial port (`/dev/ttyACM0`) is unaffected - it uses the separate `cdc_acm` driver. To make this persistent across reboots, blacklist the module:

```sh
echo "blacklist hid_mcp2200" | sudo tee /etc/modprobe.d/blacklist-mcp2200.conf
```

### Identify the MCP2200 hiddev Node

With the Tactical 1000 connected, list the available MCP2200 devices:

```sh
sudo ./mcpconfig -l
```

_Expected output (excerpt):_
```
Path: /dev/usb/hiddev4
Manufacturer: Microchip Technology Inc.
Product: MCP2200 USB Serial Port Emulator
```

Note the `Path` - `/dev/usb/hiddev4` in this example. Substitute your own path in the commands that follow.

### Record the Current GPIO State

Before changing anything, read the current GPIO register so you can restore it later:

```sh
sudo ./mcpconfig -f /dev/usb/hiddev4 -rg
```

_Expected output:_
```
01
```

A device booting normally from eMMC typically reads `0x01` or `0xc1` - both have the VCORE straps set to `0000` (eMMC FC0) with reset released, and either is fine. Write yours down so you can put it back after flashing.

### Enter Bootstrap Mode

Write the high-speed bootstrap byte twice: first with reset held low so the new VCORE straps (`1011`) latch cleanly, then with reset released so the SoC enters the TF-A FC0-HS bootstrap monitor at 921600 baud:

```sh
sudo ./mcpconfig -f /dev/usb/hiddev4 -g 0xee
sudo ./mcpconfig -f /dev/usb/hiddev4 -g 0xef
```

> **Note:** The MCP2200 GPIO write only tells the LAN9696 ROM which monitor mode to run - it does not reconfigure the host-side UART. You must also open the host serial port at 921600 baud (the next step does this through `fwu.html`). If anything is still attached at 115200, the high-speed monitor will look like garbage or silence.
>
> If you would rather stay at 115200 baud (slower flash, but simpler if a tool can't be reconfigured), use `0xe8` / `0xe9` for plain TF-A FC0 instead.

> **Important:** Before opening `fwu.html`, disconnect any serial terminal you have attached to `/dev/ttyACM*` (picocom, minicom, kermit, screen, …). The browser needs exclusive access to the serial port.

### Flash the eMMC Image with fwu.html

1. Open `fwu.html` in Chrome. It is produced by the OpenWrt build for this target and can be downloaded from https://downloads.openwrt.org/snapshots/targets/microchipsw/lan969x/trusted-firmware-a-tactical-1000/.
2. Choose the serial port that corresponds to your Tactical 1000 and connect at **921600 baud**.
3. Select the downloaded `openwrt-microchipsw-lan969x-novarq_tactical-1000-squashfs-emmc-gpt.img.gz` as the eMMC image.
4. Start the flashing procedure and wait for it to finish. Writing the full eMMC layout takes several minutes - do not unplug the device.

The tool decompresses the gzip image on the fly and writes the GPT, both FIP slots, the U-Boot environment, and the kernel/rootfs partitions in one pass.

### Return to Normal Mode

Once `fwu.html` reports a successful flash, restore the eMMC boot straps so the switch boots from eMMC on the next reset. Again, write the reset-low byte first and then the reset-released byte:

```sh
sudo ./mcpconfig -f /dev/usb/hiddev4 -g 0x00
sudo ./mcpconfig -f /dev/usb/hiddev4 -g 0x01
```

`0xc0` / `0xc1` work equivalently - they share the same eMMC VCORE straps and only differ in `GP6`/`GP7`, which do not affect boot.

Reattach your serial terminal at the default **115200 baud**. The Tactical 1000 should now go through U-Boot and boot into OpenWrt.

## First Boot and Default Networking

When OpenWrt boots for the first time you'll see the usual OpenWrt console banner followed by a root shell. There is no root password set on a fresh snapshot - set one immediately with `passwd`, especially before exposing the device to a network.

### Default Interface Names

Unlike the generic `ethN` naming used by other Linux distributions on this hardware, the OpenWrt build for the Tactical 1000 ships with friendly switchdev interface names:

- `lan1` - `lan24`: 24× Gigabit RJ-45 switch ports
- `sfp1` - `sfp4`: 4× SFP+ ports (1G/10G capable)
- `mgmt`: dedicated out-of-band management port

All of these are normal Linux netdevs - you can drive them with `ip link`, `bridge`, `ethtool`, `tc`, and so on, exactly as you would on any switchdev-backed switch.

### Default Network Configuration

Out of the box, OpenWrt bridges **every** front-panel port and the management port together into a single Layer 2 bridge called `br-lan` and assigns it a static IPv4 address:

| Item                       | Value                |
|----------------------------|----------------------|
| Bridge                     | `br-lan`             |
| Bridge members             | `lan1`-`lan24`, `sfp1`-`sfp4`, `mgmt` |
| IPv4 address               | `192.168.1.1/24`     |
| IPv6 ULA prefix            | `fd0e:fa5:9eeb::/48` (random per device) |
| DHCP / DNS                 | served on `br-lan` (dnsmasq) |

This is intentional: any port on the switch - including the dedicated management port - will give you access to the device on first boot.

Verify it from a freshly-booted device:

```sh
ip address show br-lan
```

_Expected output (excerpt):_
```
31: br-lan: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 22:c4:a6:d1:68:04 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.1/24 brd 192.168.1.255 scope global br-lan
    inet6 fd0e:fa5:9eeb::1/60 scope global noprefixroute
```

```sh
bridge link show
```

_Expected output (excerpt):_
```
2: lan4: <NO-CARRIER,BROADCAST,MULTICAST,UP> master br-lan state disabled ...
...
25: lan21: <BROADCAST,MULTICAST,UP,LOWER_UP> master br-lan state forwarding ...
30: mgmt:  <BROADCAST,MULTICAST,UP,LOWER_UP> master br-lan state forwarding ...
```

Ports without a link show as `NO-CARRIER` / `state disabled`; ports with a connected peer move to `state forwarding`.

### First Login over the Network

To log in over the network on a fresh installation:

1. Plug your laptop into any front-panel port (`lan1`-`lan24`, an SFP+ port, or the management port).
2. Configure your laptop's interface for the `192.168.1.0/24` subnet - DHCP will hand out an address automatically, or you can assign a static IP such as `192.168.1.10/24`.
3. SSH in as `root`:

```sh
ssh root@192.168.1.1
```

### Verify the Installation

A quick sanity check that the right image is running:

```sh
cat /etc/openwrt_release
```

_Expected output:_
```
DISTRIB_ID='OpenWrt'
DISTRIB_RELEASE='SNAPSHOT'
DISTRIB_TARGET='microchipsw/lan969x'
DISTRIB_ARCH='aarch64_cortex-a53'
DISTRIB_DESCRIPTION='OpenWrt SNAPSHOT ...'
```

The `DISTRIB_TARGET` line should always read `microchipsw/lan969x` on a Tactical 1000.

### Customising the Defaults

The default configuration lives in `/etc/config/network` and is managed through UCI. For example, to change the bridge IP address:

```sh
uci set network.lan.ipaddr='10.0.0.1'
uci commit network
reload_config
```

### Example: Isolated Management + Dummy L2 Switch

A common deployment turns the Tactical 1000 into a plain unmanaged-style Layer 2 switch on the front-panel ports, while keeping the dedicated management RJ-45 on its own network for administration and DHCP. We can do that with a single VLAN-aware bridge:

- **VLAN 2** - `mgmt` only, terminated on the device as `br-lan.2` with IP `192.168.1.1/24` and the OpenWrt DHCP server running on it.
- **VLAN 3** - `lan1`-`lan24` and `sfp1`-`sfp4`, no IP on the switch itself. The ports form a single broadcast domain and forward Ethernet between connected devices at wire speed - a dummy switch.

Because the two VLANs are isolated inside the bridge, traffic from the front-panel ports cannot reach the management interface and vice-versa.

The full `/etc/config/network` for this layout:

```
config interface 'loopback'
	option device 'lo'
	option proto 'static'
	list ipaddr '127.0.0.1/8'

config globals 'globals'
	option ula_prefix 'fd0e:fa5:9eeb::/48'

config device
	option name 'br-lan'
	option type 'bridge'
	list ports 'lan1'
	list ports 'lan2'
	list ports 'lan3'
	list ports 'lan4'
	list ports 'lan5'
	list ports 'lan6'
	list ports 'lan7'
	list ports 'lan8'
	list ports 'lan9'
	list ports 'lan10'
	list ports 'lan11'
	list ports 'lan12'
	list ports 'lan13'
	list ports 'lan14'
	list ports 'lan15'
	list ports 'lan16'
	list ports 'lan17'
	list ports 'lan18'
	list ports 'lan19'
	list ports 'lan20'
	list ports 'lan21'
	list ports 'lan22'
	list ports 'lan23'
	list ports 'lan24'
	list ports 'mgmt'
	list ports 'sfp1'
	list ports 'sfp2'
	list ports 'sfp3'
	list ports 'sfp4'

config interface 'lan'
	option device 'br-lan.2'
	option proto 'static'
	list ipaddr '192.168.1.1/24'
	option ip6assign '60'

config bridge-vlan
	option device 'br-lan'
	option vlan '2'
	list ports 'mgmt'

config bridge-vlan
	option device 'br-lan'
	option vlan '3'
	list ports 'lan1'
	list ports 'lan2'
	list ports 'lan3'
	list ports 'lan4'
	list ports 'lan5'
	list ports 'lan6'
	list ports 'lan7'
	list ports 'lan8'
	list ports 'lan9'
	list ports 'lan10'
	list ports 'lan11'
	list ports 'lan12'
	list ports 'lan13'
	list ports 'lan14'
	list ports 'lan15'
	list ports 'lan16'
	list ports 'lan17'
	list ports 'lan18'
	list ports 'lan19'
	list ports 'lan20'
	list ports 'lan21'
	list ports 'lan22'
	list ports 'lan23'
	list ports 'lan24'
	list ports 'sfp1'
	list ports 'sfp2'
	list ports 'sfp3'
	list ports 'sfp4'
```

Key points:

- All 29 front-panel and management ports are declared as members of `br-lan`. Per-VLAN port membership is then narrowed down by the two `config bridge-vlan` sections.
- Ports are untagged egress, so they act as plain access ports - devices plugged in do not need to know about VLANs.
- Only the management VLAN gets an IP (`br-lan.2`), so the device is reachable from `mgmt` only. The data VLAN has no L3 presence on the switch.
- The interface section is still called `lan` - the default `/etc/config/dhcp` references `interface 'lan'` for its DHCP pool, so DHCP keeps working on the management network without further edits.

To apply this configuration, write it to `/etc/config/network` and reload:

```sh
reload_config
```

You can then verify the layout:

```sh
ip address show br-lan.2
bridge vlan show
```

`br-lan.2` should carry `192.168.1.1/24`, `mgmt` should appear in VLAN 2, and the 28 data ports in VLAN 3.

## Upgrading with sysupgrade

Once OpenWrt is on the device, future updates use the standard `sysupgrade` flow with the `sysupgrade.bin` artifact. There is no need to re-enter bootstrap mode.

### Download and Apply

From a running OpenWrt system on the switch:

```sh
cd /tmp
wget https://downloads.openwrt.org/snapshots/targets/microchipsw/lan969x/openwrt-microchipsw-lan969x-novarq_tactical-1000-squashfs-sysupgrade.bin
sysupgrade /tmp/openwrt-microchipsw-lan969x-novarq_tactical-1000-squashfs-sysupgrade.bin
```

By default `sysupgrade` preserves the files listed in `/etc/sysupgrade.conf` and the contents of `/etc/config`. If you want a clean slate instead, pass `-n` to discard configuration.

The switch reboots automatically once the new kernel and rootfs are written. Reconnect over serial or management and confirm the new version with `cat /etc/openwrt_release`.

### When to Fall Back to the Full eMMC Image

`sysupgrade` only updates the kernel and rootfs partitions. Reflash the full `emmc-gpt.img.gz` through bootstrap mode if:

- The bootloader, GPT, or U-Boot environment is suspected to be damaged.
- The device fails to reach U-Boot after a failed upgrade.
- You want to return the device to a known-good factory layout.

## Troubleshooting

- **`mcpconfig -l` does not list the device** - check the USB-C cable (a data cable, not a power-only one) and confirm the host enumerates the MCP2200 with `lsusb`.
- **`fwu.html` cannot open the serial port** - make sure no other process holds `/dev/ttyACM*`, and that you are using Chrome (or another Web Serial-capable browser) over HTTPS or `file://`.
- **Flashing aborts partway** - the eMMC is left in a partial, inconsistent state and the device will likely not boot until a flash completes successfully. The SoC, however, is still in bootstrap mode (that is selected by the VCORE straps, not by anything on eMMC), so simply reopen `fwu.html` at 921600 baud and retry the flash from the beginning.
- **The switch does not boot after flashing** - re-read the MCP2200 GPIOs with `mcpconfig -rg` to confirm you have left bootstrap mode (`GP5..GP2` should read `0000`, i.e. `0x01` or `0xc1`), then power-cycle the device.

