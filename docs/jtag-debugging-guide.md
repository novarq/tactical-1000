# Novarq Tactical 1000 - JTAG Debugging Setup Guide

**Board:** Novarq Tactical 1000
**ASIC:** Microchip LAN9696 (LAN969x family, single-core ARM Cortex-A53 @ 1 GHz)
**JTAG Adapter:** Bus Blaster v4.1a (FT2232H-based, JTAGkey-compatible buffer)
**OpenOCD:** Built from source (0.12.0+dev)

---

## 1. JTAG Pin Mapping

The board uses a 10-pin ARM Cortex Debug connector. The Bus Blaster v4.1a uses
a standard 20-pin ARM JTAG header. The mapping is as follows:

| Board Pin | Board Signal | Bus Blaster 20-pin | BB Signal | Notes |
|-----------|--------------|-------------------|-----------|-------|
| 1 | 1V8 (VTref) | Pin 1 | VTref | Target voltage reference - drives BB level shifters |
| 2 | TMS | Pin 7 | TMS | JTAG mode select |
| 3 | GND | Pin 4 (or 6,8,10,12) | GND | Any even GND pin |
| 4 | TCK | Pin 9 | TCK | JTAG clock |
| 5 | GND | Pin 6 (or 4,8,10,12) | GND | Use a different GND than pin 3 |
| 6 | TDO | Pin 13 | TDO | Data out from target |
| 7 | BLIND (key) | - | - | No pin - mechanical keying |
| 8 | TDI | Pin 5 | TDI | Data in to target |
| 9 | nDETECT | GND or NC | - | 10kΩ pull-up on board, not used |
| 10 | nRSTD | Pin 15 | nSRST | Active-low system reset (see Section 7) |

**Voltage note:** The board supplies 1.8V on pin 1. The Bus Blaster's CPLD
buffer supports 1.5V–3.3V targets. **Do NOT jumper JP4** on the Bus Blaster
(which would force 3.3V) - the target must supply VTref.

**nTRST:** The board does not break out nTRST on the 10-pin connector. TAP
reset is handled via TMS sequences by OpenOCD.

---

## 2. Building OpenOCD from Source

The distro-packaged OpenOCD on Mint 22.3 may be outdated. Building from source
ensures proper AArch64 target support.

### 2.1 Install build dependencies

```bash
sudo apt update
sudo apt install -y build-essential pkg-config autoconf automake \
    libtool libusb-1.0-0-dev libftdi1-dev libhidapi-dev \
    texinfo libjaylink-dev gdb-multiarch
```

### 2.2 Clone and bootstrap

```bash
git clone https://git.code.sf.net/p/openocd/code openocd-code
cd openocd-code
git submodule init
git submodule update
./bootstrap
```

**Important:** The `git submodule` commands are required. Without them, the
bundled `jimtcl` and `libjaylink` won't be available and `./configure` will
fail with:

```
configure: error: jimtcl is required but not found via pkg-config and system includes
```

Verify submodules are present:

```bash
git submodule status
```

You should see entries for `jimtcl` and `src/jtag/drivers/libjaylink`.

### 2.3 Configure

```bash
./configure --enable-ftdi --enable-internal-jimtcl --enable-internal-libjaylink
```

The `--enable-internal-jimtcl` and `--enable-internal-libjaylink` flags are
needed because the system packages may not be available. The deprecation
warnings about internal libraries are harmless.

**Verify the configure summary shows:**

```
MPSSE mode of FTDI based devices        yes
```

If it shows `no`, `libftdi1-dev` is missing.

### 2.4 Build and install

```bash
make -j$(nproc)
sudo make install
openocd --version
```

This installs to `/usr/local/`. The binary is at `/usr/local/bin/openocd` and
scripts are at `/usr/local/share/openocd/scripts/`.

---

## 3. udev Rules and USB Setup

### 3.1 Create the udev rule for the Bus Blaster

```bash
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="0403", ATTR{idProduct}=="6010", MODE="0666", GROUP="plugdev"' \
    | sudo tee /etc/udev/rules.d/99-busblaster.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

### 3.2 Verify group membership

```bash
groups | grep plugdev
```

If `plugdev` is not listed:

```bash
sudo usermod -aG plugdev $USER
```

Log out and back in for the group change to take effect.

### 3.3 Verify USB detection

With the Bus Blaster plugged in:

```bash
lsusb | grep 0403:6010
```

Expected output:

```
Bus 00x Device 00x: ID 0403:6010 Future Technology Devices International, Ltd FT2232C/D/H Dual UART/FIFO IC
```

### 3.4 Test adapter-only (no target connected)

```bash
openocd -f interface/ftdi/dp_busblaster.cfg \
    -c "adapter speed 1000; transport select jtag; init; shutdown"
```

Expected: USB errors should NOT appear. JTAG scan failure (`all ones`) is
normal since no target is connected.

---

## 4. OpenOCD Configuration Files

The standalone configuration files are available in the [`openocd/`](openocd/) directory.

### 4.1 Interface configuration - `interface_busblaster.cfg`

```tcl
#
# Bus Blaster v4.1a - Interface Configuration
# Using default JTAGkey-compatible CPLD buffer
#
adapter driver ftdi
ftdi device_desc "Dual RS232-HS"
ftdi vid_pid 0x0403 0x6010

# JTAGkey-compatible buffer layout
ftdi layout_init 0x0c08 0x0f1b
ftdi layout_signal nTRST -data 0x0100 -noe 0x0400
ftdi layout_signal nSRST -data 0x0200 -noe 0x0800

# Conservative clock speed for reliable operation
adapter speed 1000

transport select jtag
```

### 4.2 Target configuration - `target_lan9696.cfg`

```tcl
#
# Microchip LAN9696 (LAN969x family) - Target Configuration
#
# Single-core ARM Cortex-A53 @ 1 GHz
# Novarq Tactical 1000 board
#

# Hardware SRST via nRESET (directly connected to JTAG_CPU_nRST).
# This is a full chip reset - DDR, switch core, CPU all restart.
# The boot ROM (BL1/BL2) must fully reinitialize before debug is available.
reset_config srst_only srst_nogate

# Wait 5 seconds after SRST deasserts to give BL1/BL2 time to
# reinitialize DDR and power up the debug domain.
adapter srst delay 5000

if { [info exists CHIPNAME] } {
    set _CHIPNAME $CHIPNAME
} else {
    set _CHIPNAME lan9696
}

#
# Main DAP - ARM Debug Access Port
# TAP ID 0x4ba00477 is the standard ARM CoreSight DAP ID
#
if { [info exists DAP_TAPID] } {
    set _DAP_TAPID $DAP_TAPID
} else {
    set _DAP_TAPID 0x4ba00477
}

jtag newtap $_CHIPNAME tap -irlen 4 -ircapture 0x1 -irmask 0xf \
    -expected-id $_DAP_TAPID -enable
dap create $_CHIPNAME.dap -chain-position $_CHIPNAME.tap

#
# Debug base addresses for single-core A53
# Verified working with LAN9696 on Novarq Tactical 1000.
#
set DBGBASE 0x80410000
set CTIBASE 0x80420000

# CTI - Cross Trigger Interface for the A53 core
cti create $_CHIPNAME.cti0 -dap $_CHIPNAME.dap -ap-num 0 \
    -baseaddr $CTIBASE

# Single Cortex-A53 core target
target create $_CHIPNAME.cpu0 aarch64 \
    -dap $_CHIPNAME.dap -coreid 0 \
    -dbgbase $DBGBASE -cti $_CHIPNAME.cti0

# Reset event handling:
#
# During SRST the debug power domain is off, so any DAP access will
# produce sticky errors. We suppress all DAP activity during the reset
# sequence itself, then clean up in reset-end after the delay has passed:
#
#   1. reset-assert-post:   do nothing (debug domain is unpowered)
#   2. reset-deassert-post: do nothing (may still be too early)
#   3. reset-end:           clear sticky errors, re-examine the target
#
$_CHIPNAME.cpu0 configure -event reset-assert-post {}
$_CHIPNAME.cpu0 configure -event reset-deassert-post {}
$_CHIPNAME.cpu0 configure -event examine-fail {}
$_CHIPNAME.cpu0 configure -event reset-end {
    jtag arp_init-reset
    catch { lan9696.cpu0 arp_examine }
    catch { aarch64 dbginit }
}
$_CHIPNAME.cpu0 configure -event gdb-attach { halt }

# AXI memory access port (for direct memory access without CPU)
# Neutralize reset events so the bus fabric is not disrupted.
target create $_CHIPNAME.axi mem_ap -dap $_CHIPNAME.dap -ap-num 1
$_CHIPNAME.axi configure -event reset-assert {}
$_CHIPNAME.axi configure -event reset-deassert-post {}
$_CHIPNAME.axi configure -event reset-init {}

# Default to CPU target so halt/reg/step commands go to the right place
targets $_CHIPNAME.cpu0
```

### 4.3 Board configuration - `board_novarq_tactical1000.cfg`

```tcl
#
# Novarq Tactical 1000 Board Configuration
# Microchip LAN9696 via Bus Blaster v4.1a
#
# Usage:
#   openocd -f board_novarq_tactical1000.cfg
#
source [find interface_busblaster.cfg]
source [find target_lan9696.cfg]
```

---

## 5. Running OpenOCD

### 5.1 Start OpenOCD

Power on the board, wait for U-Boot prompt on serial, then:

```bash
openocd -f interface_busblaster.cfg -f target_lan9696.cfg
```

Expected output:

```
Info : clock speed 1000 kHz
Info : JTAG tap: lan9696.tap tap/device found: 0x4ba00477 (...)
Info : lan9696.cpu0: hardware has 6 breakpoints, 4 watchpoints
Info : [lan9696.cpu0] Examination succeed
Info : Listening on port 3333 for gdb connections
```

### 5.2 Telnet interface

In another terminal:

```bash
telnet localhost 4444
```

**Important:** The default target is `lan9696.cpu0` (set in the config). If you
ever see commands going to the wrong target, check with `targets` and switch:

```
targets lan9696.cpu0
```

### 5.3 Basic debugging commands

```
halt                        # Stop the CPU
reg                         # Dump all AArch64 registers
resume                      # Resume execution
mdw 0x60000000 4            # Read 4 words from DDR base
step                        # Single step
targets                     # Show all targets and their state
```

### 5.4 Hardware reset via JTAG

```
reset
```

This triggers a full chip reset (nRESET). The board reboots through
BL1→BL2→U-Boot. After the 5-second delay, OpenOCD automatically:

1. Clears JTAG-DP sticky errors via `jtag arp_init-reset`
2. Re-examines the CPU target via `arp_examine`
3. Reinitializes debug access via `aarch64 dbginit`

The serial console will show the boot sequence during this time. Both
targets should show `running` when the reset completes.

**If `arp_examine` still fails** (timing edge case), manually recover:

```
jtag arp_init-reset
lan9696.cpu0 arp_examine
```

### 5.5 GDB debugging

With OpenOCD running, in another terminal:

```bash
gdb-multiarch
```

In GDB:

```
target extended-remote :3333
monitor halt
info registers
```

For bootloader debugging with an ELF file:

```
file /path/to/your/bootloader.elf
load
break main
continue
```

---

## 6. Serial Console via MCP2200

The Novarq Tactical 1000 uses a Microchip MCP2200 USB-to-serial bridge.

### 6.1 Serial access

The serial port appears as `/dev/ttyACM0`:

```bash
picocom -b 115200 -eb /dev/ttyACM0 --imap lfcrlf
```

### 6.2 HID access for IO configuration

The MCP2200 also exposes a USB HID interface for GPIO and configuration access.
The Linux kernel module `hid_mcp2200` claims this interface by default,
preventing userspace tools from seeing it.

**To enable userspace HID access:**

```bash
sudo rmmod hid_mcp2200
```

After this, the HID device appears at `/dev/usb/hiddevX` and tools like
`mcpconfig` can access it:

```bash
sudo ./mcpconfig -f /dev/usb/hiddev1 -i
sudo ./mcpconfig -f /dev/usb/hiddev1 -c
```

The serial port (`/dev/ttyACM0`) continues to work regardless - it uses the
`cdc_acm` driver, which is a separate USB interface.

**To make permanent** (survive reboots):

```bash
echo "blacklist hid_mcp2200" | sudo tee /etc/modprobe.d/blacklist-mcp2200.conf
```

---

## 7. Hardware Reset - nRSTD Pin Details

The JTAG connector pin 10 (`nRSTD`) is labeled `JTAG_CPU_nRST` in the
Microchip datasheet. On the Novarq Tactical 1000:

- **`JTAG_CPU_nRST` is directly connected to the chip's `nRESET` line.**
- **`nRESET` is a full chip reset** - it resets the CPU, DDR controller, switch
  fabric, and re-latches all configuration straps. This is equivalent to a
  power-on reset.
- The `reset` command from OpenOCD triggers a full cold boot (BL1→BL2→BL33/U-Boot).

### 7.1 Why reset handling is complex

After a full chip reset, the ARM CoreSight debug power domain is off. The boot
ROM (BL1) must initialize it before JTAG debug access works. This causes:

1. **"Debug regions are unpowered"** - Expected immediately after SRST.
2. **"JTAG-DP STICKY ERROR"** - The failed DAP access sets a sticky error flag
   that blocks all subsequent DAP communication until cleared.
3. **"Invalid ACK (7)"** - All JTAG lines read high because debug hardware is
   powered down.

The solution is to suppress all DAP activity during the reset sequence, wait
for the boot ROM to complete, then clear errors and re-examine:

```tcl
$_CHIPNAME.cpu0 configure -event reset-end {
    jtag arp_init-reset           ;# clear sticky errors
    catch { lan9696.cpu0 arp_examine }  ;# re-probe debug port
    catch { aarch64 dbginit }     ;# reinitialize debug
}
```

### 7.2 AXI mem_ap must not be halted during reset

The AXI memory access port (`mem_ap`) provides direct bus access. If it
participates in `reset halt`, it stalls memory transactions that the boot ROM
needs, causing BL1 to freeze. All reset events on the AXI target are
neutralized:

```tcl
$_CHIPNAME.axi configure -event reset-assert {}
$_CHIPNAME.axi configure -event reset-deassert-post {}
$_CHIPNAME.axi configure -event reset-init {}
```

### 7.3 Fallback: no hardware reset

If hardware reset causes issues, use:

```tcl
reset_config none
```

In this mode, `reset` does not toggle any hardware lines. Power cycle the board
manually for a full reset.

---

## 8. Troubleshooting Reference

### "unable to open ftdi device"

The USB driver can't access the Bus Blaster. Check:
- udev rule is in place and reloaded
- User is in the `plugdev` group (re-login after adding)
- `lsusb | grep 0403:6010` shows the device

### "JTAG scan chain interrogation failed: all ones"

No target connected, or target is not powered. Verify:
- Board is powered on
- Adapter cable is connected with correct pin 1 orientation
- JP4 on Bus Blaster is NOT jumpered

### "target not examined yet" after reset

The debug power domain wasn't ready. Run:

```
jtag arp_init-reset
lan9696.cpu0 arp_examine
```

### Commands going to wrong target

Check which target has the `*`:

```
targets
```

Switch to CPU:

```
targets lan9696.cpu0
```

### "Invalid ACK (7) in DAP response"

Debug power domain is unpowered. Wait for the board to fully boot, then:

```
jtag arp_init-reset
lan9696.cpu0 arp_examine
```

### CPU stuck in `reset` state after `reset` command

Same as above - the DAP had sticky errors. Clear them:

```
jtag arp_init-reset
lan9696.cpu0 arp_examine
halt
```

---

## 9. Quick Reference

### Start debugging session

```bash
# Terminal 1: Start OpenOCD (board must be powered and booted)
openocd -f interface_busblaster.cfg -f target_lan9696.cfg

# Terminal 2: Telnet
telnet localhost 4444

# Terminal 3: GDB (optional)
gdb-multiarch -ex "target extended-remote :3333"
```

### Common telnet workflow

```
halt                              # Stop CPU
reg                               # Show registers
mdw 0x60000000 16                 # Read memory
resume                            # Continue
reset                             # Full board reset
load_image /path/to/main.bin 0x60010000 bin   # Load binary via JTAG
```

### Verified hardware details

| Parameter | Value |
|-----------|-------|
| TAP ID | 0x4ba00477 |
| Debug base (DBGBASE) | 0x80410000 |
| CTI base (CTIBASE) | 0x80420000 |
| DDR base | 0x60000000 |
| Breakpoints | 6 hardware |
| Watchpoints | 4 hardware |
| CPU exception level (after boot) | EL1H |
| CPSR (typical) | 0x200002c5 |
| MMU | Enabled |
| D-Cache | Enabled |
| I-Cache | Enabled |
