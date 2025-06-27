# Linux-Powered VLAN Segmentation

This document will guide you to implement VLAN segmentation on Linux using the Tactical 1000's bridging capabilities. We'll transform your basic switching infrastructure into a segmented network that provides security, performance, and operational flexibility through traffic isolation.

Before implementing VLAN segmentation on your Tactical 1000, ensure you've completed the foundational network setup. If you haven't already, work through the [Initial Network Configuration Guide](initial-network-configuration-guide.md) to create your bridge interface and configure your physical ports.

## VLAN Technology Foundation

### Understanding VLAN-Aware Bridging
The Novarq Tactical 1000 supports VLAN-Aware bridging, a networking approach that enables the filtering and processing of multiple VLAN tags within network packets on bridge ports.

**Traditional Bridge Limitations:** Standard bridges operate as transparent switching devices, forwarding all traffic between all ports without consideration for logical network boundaries.

**VLAN-Aware Bridge Advantages:** VLAN-Aware bridges create logical network segments that operate independently, enabling secure multi-tenant environments, improved performance through traffic isolation, and flexible network design that adapts to organizational requirements.

### Tagged vs Untagged Frame Processing
VLAN-Aware bridges excel at handling both tagged and untagged frames simultaneously, providing seamless integration between VLAN-Aware and legacy network devices.

**Tagged Frames:** Include VLAN identification information within the frame header, enabling explicit VLAN membership designation and inter-switch VLAN communication.

**Untagged Frames:** Processed based on port-specific VLAN configuration, allowing legacy devices to participate in VLAN-segmented networks without modification.

## VLAN Configuration Implementation

### Step 1: Enable VLAN-Aware Bridge Mode
Transform your existing bridge into a VLAN-Aware switching platform:

```sh
ip link set dev br0 type bridge vlan_filtering 1
```
_If there is no output, you're all set._

**Architectural Impact:** This single command fundamentally changes your network's operational characteristics. Your bridge now operates as an switching fabric that makes forwarding decisions based on VLAN membership, creating isolated broadcast domains that enhance both security and performance.

### Step 2: Configure VLAN Segmentation Strategy
Remove ports from the default VLAN and establish intentional VLAN assignments that align with your network architecture:

**Remove default VLAN membership:**
```sh
bridge vlan del dev eth0 vid 1
bridge vlan del dev eth1 vid 1
```
_If there is no output, you're all set._

**Establish VLAN 100 membership:**
```sh
bridge vlan add dev eth0 vid 100 pvid untagged
bridge vlan add dev eth1 vid 100 pvid untagged
```
_If there is no output, you're all set._

**Configuration Analysis:**
- `vid 100` - Assigns VLAN ID 100 to these ports
- `pvid` - Designates this as the Port VLAN ID (default VLAN for untagged traffic)
- `untagged` - Removes VLAN tags when forwarding traffic to connected devices

### Step 3: Verify VLAN Configuration
Confirm your VLAN architecture implementation:

```sh
bridge vlan show
```
_Expected output:_
```
port              vlan-id  
br0               1 PVID Egress Untagged
eth0              100 PVID Egress Untagged
eth1              100 PVID Egress Untagged
```

**Configuration Validation:** This output confirms that eth0 and eth1 now operate within VLAN 100, creating an isolated network segment. Devices connected to these ports can communicate with each other but remain isolated from other VLANs, establishing the foundation for secure, segmented network operations.

## Advanced VLAN Scenarios

### Multi-VLAN Port Configuration
Create ports that participate in multiple VLANs for advanced network topologies:

```sh
# Configure trunk port with multiple VLAN access
bridge vlan add dev eth2 vid 100 tagged
bridge vlan add dev eth2 vid 200 tagged
bridge vlan add dev eth2 vid 300 pvid untagged
```
_If there is no output, you're all set._

**Advanced Configuration Impact:** This creates a trunk port that can carry traffic for multiple VLANs, enabling complex network architectures that support everything from inter-VLAN routing to advanced security policies.

### VLAN Isolation Verification

**Verify VLAN filtering status:**
```sh
cat /sys/class/net/br0/bridge/vlan_filtering
```
_Expected output:_
```
1
```

**Display comprehensive VLAN membership on eth0 interface:**
```sh
bridge vlan show dev eth0
```
_Expected output:_
```
port    vlan ids
eth0    100 PVID Egress Untagged
```

**Verify bridge forwarding database:**
```sh
bridge fdb show
```
_Expected output:_
```
01:00:5e:00:00:01 dev eth0 self permanent
33:33:00:00:00:01 dev eth0 self permanent
aa:bb:cc:dd:ee:ff dev eth0 vlan 100 master br0 permanent
bb:cc:dd:ee:ff:aa dev eth0 vlan 100 master br0
```

**Network Segmentation Validation:**
These commands provide visibility into your VLAN architecture's operational status, confirming that your segmentation strategy is functioning as designed.

## Foundation for Advanced Networking

Your VLAN-Aware bridge configuration establishes the foundation for sophisticated network architectures that combine security, performance, and operational flexibility. This implementation demonstrates how the Tactical 1000's advanced capabilities transform basic connectivity into intelligent infrastructure that adapts to evolving business requirements.
