# Selective Routing & Wireguard

## Overview

This guide shows how to configure VLAN segmentation on the Tactical 1000 with WireGuard, separating direct internet traffic from secure VPN tunnels across distinct network ports.

**Core Architecture:**
- VLAN 100: Direct Internet access via physical uplink
- VLAN 200: Secure tunneling via WireGuard tunnel
- Complete isolation between VLANs with policy-based routing

The Tactical 1000's Linux foundation delivers enterprise-grade networking through standard tools, making advanced segmentation accessible to everyone.

## Prerequisites

- Novarq Tactical 1000 powered by Linux and with WireGuard preinstalled
- Administrative access to the device

### Network Interface Architecture

The Tactical 1000's port design creates clear traffic separation through purposeful interface allocation:

**Switch Ports (`eth1`, `eth2`)** - These ports connect client devices and become part of the managed bridge infrastructure. Traffic from these ports flows through the VLAN-aware bridge, where intelligent routing decisions direct packets to their appropriate destinations.

**Uplink Port (`eth28`)** - This dedicated interface serves as the direct gateway to external networks. By maintaining separation from the bridge infrastructure, eth28 ensures clean routing for VLAN 100 traffic that requires unencumbered internet access.

### Traffic Flow Design

Set up clients as follows:

```
Device on eth1 (VLAN 100):
├── IP: 192.168.1.100/24 (must be within 192.168.1.0/24 subnet)
├── Gateway: 192.168.1.10
└── Traffic Path: eth1 → br0.100 → eth28 → Internet

Device on eth2 (VLAN 200):
├── IP: 192.168.2.100/24 (must be within 192.168.2.0/24 subnet)
├── Gateway: 192.168.2.10
└── Traffic Path: eth2 → br0.200 → wg0 → VPN Server → Internet
```

Each client device requires IP configuration that matches its designated VLAN subnet. This guide uses static IP addresses for clarity and simplicity, but implementing DHCP servers on each VLAN interface would automate client configuration. This is a good exercise left to the reader for enhanced deployment flexibility.

## System Preparation

If WireGuard is not already installed on your system, you can follow these example steps for Ubuntu/Debian-based distributions. Note that installation methods vary by distribution, and these commands are provided for reference only:

```bash
# Install essential package management tools
apt install -y software-properties-common

# Enable universe repository for extended package access
add-apt-repository universe

# Update package database
apt update

# Install WireGuard
apt install -y wireguard
```

### Client Configuration Requirements

Sample static IPv4 configuration on the client devices:

**VLAN 100 Device (Direct Internet Access):**
```
IP Address: 192.168.1.100
Subnet Mask: 255.255.255.0
Gateway: 192.168.1.10
DNS: 8.8.8.8
```

**VLAN 200 Device (Secure Tunnel Access):**
```
IP Address: 192.168.2.100
Subnet Mask: 255.255.255.0
Gateway: 192.168.2.10
DNS: 1.1.1.1
```

### Step 1: Create and Configure Bridge

The bridge connects switch ports into a single manageable interface for VLAN traffic handling. For comprehensive understanding of bridge fundamentals, reference the [Initial Network Configuration Guide](initial-network-configuration-guide.md) which details the Tactical 1000's networking architecture.

```bash
# Create bridge interface
ip link add name br0 type bridge

# Bring up the bridge interface
ip link set br0 up

# Add eth1 and eth2 to bridge (uplink eth28 remains independent)
ip link set dev eth1 up
ip link set eth1 master br0

ip link set dev eth2 up
ip link set eth2 master br0
```

The uplink port (`eth28`) operates independently from the bridge, ensuring clear separation between switching and routing functions.

### Step 2: Enable VLAN-Aware Bridging

Here we move from the basic bridge into an intelligent VLAN-aware system that provides granular traffic control. This implementation follows the principles outlined in [Linux-Powered VLAN Segmentation](linux-powered-vlan-segmentation.md), demonstrating how the Tactical 1000 leverages Linux's advanced networking capabilities for enterprise-grade segmentation:

```bash
# Enable VLAN-aware bridge mode
ip link set dev br0 type bridge vlan_filtering 1

# Configure VLAN 100 for eth1 (direct internet path)
bridge vlan del dev eth1 vid 1 2>/dev/null
bridge vlan add dev eth1 vid 100 pvid untagged

# Configure VLAN 200 for eth2 (secure tunnel path)
bridge vlan del dev eth2 vid 1 2>/dev/null
bridge vlan add dev eth2 vid 200 pvid untagged

# Register br0 with the bridge VLAN table
bridge vlan add vid 100 dev br0 self
bridge vlan add vid 200 dev br0 self
```

This configuration enables automatic VLAN assignment at the port level. Traffic entering eth1 is tagged as VLAN 100, while eth2 traffic is tagged as VLAN 200. The bridge handles VLAN tagging/untagging transparently, eliminating the need for VLAN configuration on client devices.

### Step 3: Initialize WireGuard Interface

The Tactical 1000 supports multiple approaches for WireGuard initialization:

- Standard Configuration: `wg-quick up wg0` using system configuration files
- Service Management: `systemctl start wg-quick@wg0` for persistent operation
- Manual Configuration: Direct interface creation for custom implementations
- Automated Scripts: Integration with existing network automation

Ensure your WireGuard interface `wg0` is operational and properly configured before proceeding. Verify tunnel status with the following command:

```bash
# Check WireGuard interface status and connectivity
wg show wg0
```
_Expected output:_
```
interface: wg0
  public key: [HIDDEN]
  private key: (hidden)
  listening port: 50025
  fwmark: 0xca6c

peer: [HIDDEN]
  preshared key: (hidden)
  endpoint: [HIDDEN]:51820
  allowed ips: 0.0.0.0/0, ::/0
  latest handshake: 4 seconds ago
  transfer: 620 B received, 644 B sent
```

This command displays essential tunnel information including peer connections, data transfer statistics, and endpoint status. The Tactical 1000's architecture accommodates various deployment scenarios while maintaining security standards—transforming complex VPN management into transparent, manageable operations.

### Step 4: Create VLAN Interfaces and Routing Tables

Establish the logical network layers that enable intelligent traffic direction:

```bash
# Create VLAN interfaces on the bridge
ip link add link br0 name br0.100 type vlan id 100
ip link add link br0 name br0.200 type vlan id 200

# Configure gateway addresses for each VLAN
ip addr add 192.168.1.10/24 dev br0.100
ip addr add 192.168.2.10/24 dev br0.200

# Activate the VLAN interfaces
ip link set br0.100 up
ip link set br0.200 up

# Create dedicated routing tables for policy-based routing
echo "100 vlan100" >> /etc/iproute2/rt_tables
echo "200 vlan200" >> /etc/iproute2/rt_tables
```

### Step 5: Configure Policy-Based Routing

Implement routing policies that automatically direct traffic through appropriate network paths:

```bash
# VLAN 100 - Direct internet access routing
ip route add 192.168.1.0/24 dev br0.100 src 192.168.1.10 table vlan100
ip route add default dev eth28 table vlan100

# VLAN 200 - Secure tunnel routing
ip route add 192.168.2.0/24 dev br0.200 src 192.168.2.10 table vlan200
ip route add default dev wg0 table vlan200

# Implement policy-based routing rules
ip rule add from 192.168.1.0/24 table vlan100 priority 100
ip rule add from 192.168.2.0/24 table vlan200 priority 200
```

### Step 6: Configure Security and Isolation

Deploy security policies that ensure network isolation while enabling appropriate connectivity:

```bash
# Enable packet forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Configure Network Address Translation
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth28 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.2.0/24 -o wg0 -j MASQUERADE

# Enable bidirectional traffic flow
iptables -A FORWARD -i br0.100 -o eth28 -j ACCEPT
iptables -A FORWARD -i eth28 -o br0.100 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i br0.200 -o wg0 -j ACCEPT
iptables -A FORWARD -i wg0 -o br0.200 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Enforce complete VLAN isolation
iptables -A FORWARD -i br0.100 -o br0.200 -j DROP
iptables -A FORWARD -i br0.200 -o br0.100 -j DROP
```

## Verification and Monitoring

### Verify VLAN configuration
```bash
bridge vlan show
```
_Expected output:_
```
port              vlan-id  
eth1              100 PVID Egress Untagged
eth2              200 PVID Egress Untagged
br0               1 PVID Egress Untagged
                  100
                  200
```

### Check specific port assignments
```bash
bridge vlan show dev eth1
bridge vlan show dev eth2
```
_Expected output:_
```
port              vlan-id  
eth1              100 PVID Egress Untagged
port              vlan-id  
eth2              200 PVID Egress Untagged
```

### Validate routing tables
```bash
ip route show table vlan100
ip route show table vlan200
```
_Expected output:_
```
default via 192.168.8.1 dev eth28 
192.168.1.0/24 dev br0.100 scope link src 192.168.1.10 
default dev wg0 scope link 
192.168.2.0/24 dev br0.200 scope link src 192.168.2.10
```

### Review routing policies
```bash
ip rule show
```
_Expected output:_
```
0:	from all lookup local
100:	from 192.168.1.0/24 lookup vlan100
200:	from 192.168.2.0/24 lookup vlan200
32764:	from all lookup main suppress_prefixlength 0
32765:	not from all fwmark 0xca6c lookup 51820
32766:	from all lookup main
32767:	from all lookup default
```

### Monitor WireGuard status
```bash
wg show wg0
```
_Expected output:_
```
interface: wg0
  public key: [HIDDEN]
  private key: (hidden)
  listening port: 50025
  fwmark: 0xca6c

peer: [HIDDEN]
  preshared key: (hidden)
  endpoint: [HIDDEN]:51820
  allowed ips: 0.0.0.0/0, ::/0
  latest handshake: 4 seconds ago
  transfer: 620 B received, 644 B sent
```

### Verify interface states
```bash
ip link show | grep -E "(br0|wg0|eth1|eth2)"
```
_Expected output:_
```
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP mode DEFAULT group default qlen 1000
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP mode DEFAULT group default qlen 1000
31: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
32: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
33: br0.100@br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
34: br0.200@br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
```

## Security Architecture

This configuration delivers enterprise-grade network security through:

* Complete Network Isolation: VLANs operate as independent network segments with no cross-communication capabilities, ensuring that sensitive and public traffic never intersect.

* Intelligent Traffic Direction: Policy-based routing automatically directs traffic through appropriate channels without requiring client configuration, simplifying deployment while maintaining security.

* Comprehensive Access Control: Firewall policies provide granular control over traffic flow while enabling legitimate connectivity patterns.

* Secure Tunnel Integration: WireGuard integration provides military-grade encryption for sensitive communications while maintaining transparent operation.
