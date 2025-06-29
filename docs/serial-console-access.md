# Serial Console Access Guide

## Introduction

The Novarq Tactical 1000 switch provides a serial console interface for initial configuration, troubleshooting, and emergency management or when network access is unavailable. This guide walks you through establishing a serial console connection to your switch, enabling you to perform essential configuration tasks and system diagnostics.

## Physical Setup

Connect your cable between your computer and the USB-C console port on the front panel of your switch.

Before proceeding, ensure the switch is powered on. Once configured, the serial console will display output from the moment power is applied, including the bootloader and entire boot process.

## Software Setup

Depending on your OS, you need to identify your serial device:

- On Linux/macOS: Usually `/dev/ttyACM0` 
- On Windows: Check Device Manager for the COM port number

You may use any serial terminal connection software to configure the switch, as long as it supports standard serial communication settings.

Below are some common options:

### Method 1: Using C-kermit

Rather than manual configuration for each session, we establish a standardized, reusable configuration that ensures consistent connectivity.

#### Create Configuration File
```bash
vi novarq_kermit
```
#### Configuration Content
```
# Novarq Tactical 1000 switch serial configuration
set line /dev/ttyACM0
set speed 115200
set carrier-watch off
set flow-control none
set handshake none
set prefixing all
set streaming off
set parity none

# Auto-connect when script is loaded
connect
```
#### Launch Console
```bash
kermit novarq_kermit
```
To disconnect from the session, use `Ctrl+\` followed by `C` to return to the kermit prompt.

#### Method 2: Using minicom 

```bash
# Configure minicom
minicom -s

Serial Device: /dev/ttyACM0
Baud Rate: 115200
Data Bits: 8
Stop Bits: 1
Parity: None
Hardware Flow Control: No
Software Flow Control: No

Save as: novarq_minicom (creates reusable profile)
```

```bash
# Connect to the switch
minicom novarq_minicom
```

To exit minicom, press `Ctrl+A` followed by `X`.

### Method 3: Using PuTTY

1. **Launch PuTTY** and configure the connection:
   - Connection type: Serial
   - Serial line: COM port number (e.g., COM3)
   - Speed: 115200
   - Data bits: 8
   - Stop bits: 1
   - Parity: None
   - Flow control: None
2. **Click "Open"** to establish the connection

## Initial Login and Configuration

Once connected, you may need to press Enter to see the console output. What you see depends on the switch's current state - it may show boot messages, a login prompt, or an existing session.

You’re now ready to configure your switch. Whether it’s your first setup or a recovery operation, access to the serial console gives you the control you need — clearly, simply, and reliably.
