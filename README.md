# Tactical 1000 🚀

The _Tactical 1000_ is Novarq's first switch, combining the performance of enterprise-grade hardware with the freedom and transparency of open-source software. Built on the Microchip LAN9696 ASIC and powered by pure Linux networking, it delivers wire-speed performance with complete control and flexibility.

## 🌟 Why Tactical 1000?

- **💎 Open and Proven**: Native Linux architecture through switchdev framework
- **🛡️ Network Sovereignty**: Full ownership and control over your entire network operation
- **⚡ Blazing-Fast Switching**: Non-blocking design delivers consistent wire-speed performance
- **🐧 Your Linux, Your Way**: Deploy OpenWrt, Debian, Ubuntu, or any distro that fits your needs
- **🌍 Future-Proof**: Mainline Linux kernel support and community-driven development
- **🛠️ Build or Accelerate**: Customize from scratch or leverage our proven implementations

## 📋 Technical Specifications

- **Platform**: Microchip [LAN9696TSN-V/3KW](https://www.microchip.com/en-us/product/lan9696)
- **Switching Capacity**: 66 Gbit/s non-blocking
- **Data Connectivity**
  - 24x RJ-45 10/100/1000BASE-T ports
  - 4x SFP+ ports (1G/10G capable)
- **Management & Console**
  - 1x dedicated Out-Of-Band management port
  - 1x USB Type-C console port
  - 1x USB Type-A host port
- **Form Factor:** Standard 1U rack-mountable design
- **CPU**: 64-bit ARM Cortex-A53, single-core @ 1 GHz
- **DRAM**: 2GB DDR4
- **Storage**: 16GB eMMC NAND + 256MB SPI NAND + 32MB QSPI NOR
- **PoE Support**: 🚧 Coming soon

## ⚙️ Features

**Switching Performance**
- Full wire-speed, non-blocking switching up to 10 Gbps per port
- Advanced programmable TCAM classification
- 12 Mb of integrated shared packet memory
- Low-latency cut-through forwarding mode
- Energy Efficient Ethernet (IEEE 802.3az)

**Layer 2 Switching**
- High-capacity L2 switching handles enterprise-scale networks
- 16K non-hashed MAC address table
- 4K VLANs with IEEE 802.1Q C-tag and S-tag support

**Layer 3 Routing**
- Hardware-accelerated routing delivers wire-speed performance without software bottlenecks
- IPv4/IPv6 unicast and multicast forwarding with reverse path forwarding

**Configuration management and telemetry**
- NETCONF/YANG (Netopeer2 and sysrepo) based support for datastore configuration management and telemetry

**Quality of Service**
- Intelligent traffic prioritization ensures critical applications get the bandwidth they need
- 8 priority queues per port with strict priority and DWRR scheduling
- TCAM-based classification with Layer 2-4 pattern matching
- Dual-rate policers and flexible QoS mapping (1K ingress, 2K egress)
- Priority-based flow control (IEEE 802.1Qbb)

**Security & Access Control**
- Hardware-based Access Control Lists (ACLs)
- Advanced ingress and egress security enforcement
- Port-based security with pattern matching capabilities
- Comprehensive traffic policing and rate limiting

### 🛠️ Software Stack

The Tactical 1000 runs on a pure Linux foundation:

- **Native Linux tools**: Standard `ip`, `bridge`, `tc` commands
- **Switchdev Framework**: Hardware offloading with standard Linux interfaces
- **Mainline Support**: Full kernel integration and without proprietary drivers 

## 🏗️ Architecture

The Tactical 1000 breaks free from proprietary limitations with a pure Linux foundation that puts you in complete control:

```
┌───────────────────────────────────────────────────────────────┐
│                      Your Applications                        │
│           Deploy anything: OpenWrt, Debian, Ubuntu...         │
├───────────────────────────────────────────────────────────────┤
│                   Pure Linux Network Stack                    │
│     Standard tools you already know: ip, bridge, tc, etc.     │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   SSH/CLI   │  │   Web GUI    │  │   NETCONF/RESTCONF   │  │
│  │   Native    │  │   Optional   │  │      Standards       │  │
│  └─────────────┘  └──────────────┘  └──────────────────────┘  │
├───────────────────────────────────────────────────────────────┤
│                   Linux Switchdev Framework                   │
│          Hardware acceleration with zero vendor lock-in       │
├───────────────────────────────────────────────────────────────┤
│                   Open Source LAN9696 Driver                  │
│            Mainline kernel support - no binary blobs          │
├───────────────────────────────────────────────────────────────┤
│           Microchip LAN9696 ASIC - 66 Gbps non-blocking       │
│               wire-speed performance you can trust            │
└───────────────────────────────────────────────────────────────┘
```


## 🏢 Use Cases

### Enterprise Networks
- **Campus switches** connecting multiple buildings and departments with advanced traffic prioritization
- **Security-conscious deployments** requiring complete transparency
- **Management network switches** for data center and server infrastructure

### Telecommunications & Service Providers
- **Point of Presence (PoP)** for carrier-grade service delivery and traffic aggregation
- **Service provider edge** deployments requiring flexible traffic engineering and policy enforcement
- **Wholesale network services** enabling white-label switching solutions for partners

### Government & Critical Infrastructure
- **Supply chain independence** including hardware and software
- **Security compliance** through transparent implementations
- **Long-term support** via community maintenance

### Development & Research
- **Network protocol development** with kernel-level access for unrestricted research capabilities
- **Open networking education** with full source availability

## 🐧 Operating systems
- [**Buildroot**](https://github.com/novarq/buildroot-external-novarq)
- [**Debian**](https://github.com/novarq/debootstrap)
- [**Ubuntu**](https://github.com/novarq/debootstrap)
- OpenWrt 🚧

## 📚 Guides and Tutorials
- [**Serial Console Setup Guide**](docs/serial-console-access.md)
- [**Initial Network Configuration Guide**](docs/initial-network-configuration-guide.md)
- [**Linux-Powered VLAN Segmentation**](docs/linux-powered-vlan-segmentation.md)
- [**NETCONF based Network Configuration Guide**](docs/netconf-configuration-guide.md)
- [**Docker Setup Guide**](docs/docker-setup-guide.md)
- [**Selective Routing & Wireguard**](docs/selective-routing-and-wireguard.md)

## 📦 Getting Hardware

Ready to experience open networking? We make it simple to get started.

The Tactical 1000 is available through Novarq's. Contact us for:
- **Evaluation Units**: For testing and development
- **Volume Pricing**: Enterprise and education discounts
- **Custom Configurations**: Specialized requirements
- **Support Contracts**: Professional support options

**Contact**: [sales@novarq.com](mailto:sales@novarq.com)

---
<p align="center">Built with ❤️ by the Novarq team</p>
