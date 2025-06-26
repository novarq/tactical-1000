# Initial Network Configuration Guide

## Getting Your Network Infrastructure Up and Running

This guide walks you through the essential steps to configure your Novarq Tactical 1000 switch running Linux. We've designed this to be straightforward, practical, and Linux distribution agnostic. Here you'll find the configuration steps you need to get your network operational quickly and efficiently, regardless of your Linux environment.

## Understanding Your Ports

The Tactical 1000 provides three types of network interfaces:

**Switching Ports:**
- _eth0_ - _eth23_: 24x Gigabit Ethernet RJ-45 ports for connecting your devices
- _eth24_ - _eth27_: 4x SFP+ ports for 1G/10G fiber connections

**Management Port:**
- _eth28_: Dedicated out-of-band management port

**Console Port:**
- USB-C: For initial configuration via serial console (see our [Serial Console Guide](serial-console-access.md))

Let's see how these appear on your system. The Tactical 1000's interfaces follow a simple naming convention that makes them easy to identify and manage:

#### Check current interface status
```sh
ip a
```
_Expected output:_
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether c6:9f:69:b4:60:01 brd ff:ff:ff:ff:ff:ff
    altname enxc69f69b46001

.
.
.
30: eth28: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether c6:9f:69:b4:60:1e brd ff:ff:ff:ff:ff:ff
    altname enxc69f69b4601e
```

All 29 network interfaces are available on your switch - _eth0_ through _eth27_ for data switching, plus eth28 for dedicated management. Each interface has a unique MAC address and is ready to be configured for your network's needs.

## Basic Network Configuration

### Step 1: Initialize Management Connectivity

Connect the dedicated out-of-band management port (_eth28_) to your network using an Ethernet cable. How you configure this interface, whether through DHCP from your router or with a static IP, is up to you and your network requirements. This is standard Linux networking, so use whatever method fits your environment best.

#### Bring up the management interface
```sh
ip link set eth28 up
```
_Expected output:_
```
[   99.643767] sparx5-switch e00c0000.switch eth28: PHY [e20101a8.mdio-mii:03] driver [Microchip LAN8841 Gigabit PHY] (irq=19)
[   99.667554] sparx5-switch e00c0000.switch eth28: configuring for phy/rgmii link mode
[  102.887383] sparx5-switch e00c0000.switch eth28: Link is Up - 1Gbps/Full - flow control off
```
This brings your management interface online and connects it to your network. The service port (eth28) can be used to configure and monitor the switch remotely.

### Step 2: Verify Internet Connectivity

#### Test connectivity
```sh
ping 1.1.1.1 -c1
```
_Expected output:_
```
PING 1.1.1.1 (1.1.1.1): 56 data bytes
64 bytes from 1.1.1.1: seq=0 ttl=55 time=24.592 ms

--- 1.1.1.1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
```

This confirms your switch can reach the internet, ensuring you have the connectivity needed for updates, monitoring, and remote management.

### Step 3: Configure Layer 2 Switching

Now let's set up the core switching functionality that makes your network infrastructure work. The Tactical 1000's hardware-accelerated switching delivers wire-speed performance through the switchdev framework.

#### Create bridge for switching
```sh
ip link add name br0 type bridge
```
_If there is no output, you're all set._
```sh
ip link set br0 up
```
_If there is no output, you're all set._

The above will create the bridge interface that enables Layer 2 switching between your connected devices, leveraging the Tactical 1000's 66 Gbps non-blocking switching capacity.

### Step 4: Enable and Configure Switch Ports

#### Bring up physical ports (examples with eth0 and eth1)
```sh
ip link set dev eth0 up
```
_If there is no output, you're all set._
```sh
ip link set dev eth1 up
```
_If there is no output, you're all set._
#### Add ports to bridge for switching
```sh
ip link set eth0 master br0
```
_If there is no output, you're all set._
```sh
ip link set eth1 master br0
```
_If there is no output, you're all set._

The above commands will activate your switch ports and connects them to the bridge, enabling devices plugged into these ports to communicate with each other at wire speed.

Once you have physically connected devices to the switch, you can check that the configuration has been set properly and is working correctly:

#### Check bridge member interfaces
```sh
ip link show master br0
```
_Expected output:_
```sh
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP group default qlen 1000
    link/ether c6:9f:69:b4:60:01 brd ff:ff:ff:ff:ff:ff
    altname enxc69f69b46001
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP group default qlen 1000
    link/ether c6:9f:69:b4:60:02 brd ff:ff:ff:ff:ff:ff
    altname enxc69f69b46002
```
#### View bridge details
```sh
bridge link show
```
_Expected output:_
```
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 5
eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 5
```

This shows that your ports are active (UP), connected to the bridge (master br0), and in forwarding state—ready to pass traffic between connected devices at full wire speed.

## Expanding Your Configuration

The Tactical 1000 provides 24 Gigabit Ethernet ports plus 4 SFP+ ports. Add more ports as needed:

```sh
ip link set dev eth2 up
ip link set eth2 master br0
```

## Configuration Persistence

The commands in this guide configure your switch for the current runtime session. Like all standard Linux networking, these settings will reset after a reboot unless you make them persistent through your distribution's network configuration system.

Persist this configuration using whatever method works best for you: _systemd-networkd_, _/etc/network/interfaces_, _NetworkManager_, or your preferred approach. The Tactical 1000 works out of the box with your existing Linux tools and workflows.

## What's Next?

This gets your basic switching functionality up and running, leveraging the Tactical 1000's enterprise-grade performance with open-source flexibility. Your network infrastructure should work for you. The Tactical 1000 gives you the foundation to build a simple and reliable network that is ready to scale with complete transparency and control.
