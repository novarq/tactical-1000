# BIRD Installation and Configuration Guide

## Overview

This guide sets up IPv4 and IPv6 BGP routing with BIRD on a Tactical 1000 running Debian or Ubuntu. BIRD learns routes from neighboring routers and installs them into the Linux routing table, and the LAN969x/Sparx5 routing driver then programs eligible routes into the switch ASIC. No separate forwarding agent is needed.

The example routes traffic between two VLANs on different IP networks. BIRD only selects the routes. The ASIC forwards the packets: it performs the Layer 3 lookup, decrements the TTL or hop limit, and rewrites the Ethernet addresses.

## Prerequisites

- Debian or Ubuntu (arm64) installed on the Tactical 1000, with working management connectivity
- A LAN969x/Sparx5 kernel with Layer 3 routing support
- Root access through the management port or the [serial console](serial-console-access.md)
- Two unused switch ports and two external BGP peers
- No other network manager or routing daemon configuring the ports and routing table used here

Installing BIRD does not add hardware routing support; that comes from the kernel driver. For general bridge and VLAN setup, see the [Initial Network Configuration Guide](initial-network-configuration-guide.md) and [Linux-Powered VLAN Segmentation](linux-powered-vlan-segmentation.md). This example creates a separate bridge and leaves any existing `br0` configuration untouched.

Keep the dedicated management port out of the bridge, and do not change its addresses or default route. Back up your existing network and BIRD configuration before you start, and run all commands as root.

## Example Network

Each switch port carries untagged traffic in its own VLAN, and VLAN interfaces on the bridge hold the router addresses.

| Link | Switch port / VLAN | Tactical 1000 address | Peer address |
| --- | --- | --- | --- |
| Left IPv4 | `sfp1` / 110 | `192.0.2.1/24` | `192.0.2.2/24` |
| Left IPv6 | `sfp1` / 110 | `2001:db8:110::1/64` | `2001:db8:110::2/64` |
| Right IPv4 | `sfp2` / 120 | `198.51.100.1/24` | `198.51.100.2/24` |
| Right IPv6 | `sfp2` / 120 | `2001:db8:120::1/64` | `2001:db8:120::2/64` |

The Tactical 1000 uses AS 65000, and the left and right peers use AS 65001 and AS 65002. All addresses, AS numbers, VLAN IDs, and port names are examples; replace them to suit your network.

Check your interface names before you continue:

```sh
ip -br link
ip -br address
bridge link show
```

The examples use `sfp1` and `sfp2`; substitute your image's port names throughout. Both ports must be unconfigured and not part of any existing bridge, and no `br-bird` bridge should exist yet. Keep the two peer networks separate, even when the peers are virtual machines on the same host, so that traffic between them has to cross the Tactical 1000.

## Installation Steps

### 1. Install BIRD and Network Tools

Update the package index and check which BIRD packages your release provides:

```sh
apt-get update
apt-cache policy bird2 bird3
```

Where BIRD 2 is available, install it along with some network diagnostic tools:

```sh
apt-get install bird2 iproute2 iputils-ping ethtool tcpdump
```

If your release provides BIRD 3 and you prefer it, install `bird3` instead. On Ubuntu, the BIRD packages are in the Universe repository, so enable it if `apt-cache` finds nothing. If BIRD is already installed and its version meets your needs, keep it. Check the version and the service definition:

```sh
bird --version
systemctl cat bird.service
```

The configuration below uses the channel syntax shared by BIRD 2 and 3; it does not work with BIRD 1. The examples assume the Debian and Ubuntu defaults, `bird.service` and `/etc/bird/bird.conf`. If your package's service definition uses different names, adjust the commands to match. Do not run two BIRD instances against the same routing tables.

Stop BIRD while you set up the example:

```sh
systemctl stop bird.service
```

### 2. Check the Kernel Configuration

You need a Tactical 1000 kernel that includes the LAN969x/Sparx5 Layer 3 routing driver; Novarq maintains the sources in [novarq/linux](https://github.com/novarq/linux). The kernel must also be built with `CONFIG_IP_ROUTE_MULTIPATH=y`, because the driver's nexthop-weight checks can depend on it even for single-path IPv4 routes. Check the running kernel's configuration:

```sh
uname -r
if [ -r /proc/config.gz ]; then
  zcat /proc/config.gz | grep '^CONFIG_IP_ROUTE_MULTIPATH='
else
  grep '^CONFIG_IP_ROUTE_MULTIPATH=' "/boot/config-$(uname -r)"
fi
```

The output should be:

```text
CONFIG_IP_ROUTE_MULTIPATH=y
```

This is a kernel build option, not a BIRD setting or a sysctl. If the running kernel's configuration is not available on the device, get it from your image provider. Your kernel's driver documentation lists the routing features it supports.

The kernel revision and board configuration come from the [Novarq Buildroot external tree](https://github.com/novarq/buildroot-external-novarq). The [Novarq Debian/Ubuntu build workflow](https://github.com/novarq/debootstrap) builds the root filesystem with debootstrap and pairs it with a Buildroot-built kernel.

If the option is disabled, rebuild the kernel. Set up a build by following the external tree's instructions. Then, in the configured Buildroot directory on your build host, run `make linux-menuconfig`, search for `IP_ROUTE_MULTIPATH`, enable it, save, and exit. Run `make` to rebuild. Make sure the change ends up in your build inputs so that a clean rebuild keeps it; if the build uses a custom kernel configuration file, `make linux-update-defconfig` writes the change back to that file.

Deploy `output/images/uImage-lan969x.itb` and any matching kernel modules by following your image's deployment instructions, boot the new kernel, and run the check again.

### 3. Configure the VLAN-Aware Routing Interfaces

This setup places the router addresses on VLAN interfaces above a VLAN-aware bridge. Addresses assigned directly to physical ports follow a different hardware routing model, which this guide does not cover.

Create the bridge and attach the two data ports:

```sh
ip link add br-bird type bridge vlan_filtering 1 vlan_default_pvid 0 \
  stp_state 0 mcast_snooping 1
ip link set br-bird up

ip link set sfp1 master br-bird
bridge vlan add dev sfp1 vid 110 pvid untagged
bridge vlan add dev br-bird vid 110 self
ip link set sfp1 up

ip link set sfp2 master br-bird
bridge vlan add dev sfp2 vid 120 pvid untagged
bridge vlan add dev br-bird vid 120 self
ip link set sfp2 up

# Set the router MAC after attaching the switch ports.
ip link set br-bird address 02:00:00:00:00:01
```

Set the bridge MAC address only after the switch ports are attached, so that the driver updates the router MAC in the ASIC. Keep this order in any persistent configuration as well. Choose a locally administered MAC address that is unique in your network; the one above is only an example.

Create the routed VLAN interfaces:

```sh
ip link add link br-bird name v110 type vlan id 110
ip address add 192.0.2.1/24 dev v110
ip -6 address add 2001:db8:110::1/64 dev v110
ip link set v110 up

ip link add link br-bird name v120 type vlan id 120
ip address add 198.51.100.1/24 dev v120
ip -6 address add 2001:db8:120::1/64 dev v120
ip link set v120 up
```

To enable IPv4 and IPv6 forwarding, create `/etc/sysctl.d/90-bird-forwarding.conf` with:

```ini
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

Apply it:

```sh
sysctl -p /etc/sysctl.d/90-bird-forwarding.conf
```

Once IPv6 forwarding is enabled, Linux stops accepting Router Advertisements on interfaces that use the default `accept_ra=1`. If the management port gets its IPv6 address or default route from Router Advertisements, configure it to keep accepting them, for example by adding `net.ipv6.conf.<mgmt>.accept_ra=2` to the same file, where `<mgmt>` is the management interface name. If a network manager configures the management port, use its equivalent setting instead.

### 4. Configure IPv6 Neighbor Discovery

With the current Tactical 1000 driver, IPv6 neighbor discovery needs one extra setting. When an IPv6 address is added to a VLAN interface, the kernel announces the address's solicited-node multicast group in an MLD report, and the bridge records it as a host multicast membership. The driver copies multicast to the CPU only for groups with such a membership. The memberships expire after the bridge's multicast membership interval, 260 seconds by default, unless an MLD querier renews them, and point-to-point links usually have no querier. After that, neighbor solicitations for the Tactical 1000's addresses no longer reach Linux. Neighbors that are already talking to the switch keep working, but a neighbor that has to resolve it again, for example after a reboot, fails, and BGP sessions over IPv6 can drop. IPv4 is not affected, because ARP uses broadcast, which always reaches the CPU.

Keep the memberships in place with one of the two methods below. The per-VLAN querier needs no list of groups; the refresh timer does not make the switch send queries.

#### Option 1: Per-VLAN MLD Querier

Make the bridge the MLD querier in each routed VLAN:

```sh
ip link set dev br-bird type bridge mcast_vlan_snooping 1
bridge vlan global set dev br-bird vid 110 mcast_querier 1
bridge vlan global set dev br-bird vid 120 mcast_querier 1
```

The bridge then sends an MLD query into each VLAN every 125 seconds. The VLAN interfaces answer it like any other host, and their reports renew the memberships for every group they have joined, so nothing needs updating when the addresses change. Enable the querier per VLAN, after the VLANs are on the bridge: a bridge-wide `mcast_querier 1` sends untagged queries, which never reach the VLAN interfaces. The bridge sends the queries from its own IPv6 link-local address, which it gets automatically once it is up.

Hosts in these VLANs also receive the queries and answer them, which adds a few small packets every two minutes. How the bridge interacts with another MLD querier in the same VLAN has not been tested.

Check the result:

```sh
bridge vlan global show dev br-bird
bridge -s mdb show dev br-bird
```

VLANs 110 and 120 should show `mcast_querier 1`, and the host entries (`port br-bird`) should stay listed instead of expiring after 260 seconds.

#### Option 2: Refresh Timer

Alternatively, add the memberships yourself and refresh them before they expire. Create `/usr/local/sbin/bird-nd-refresh` with:

```sh
#!/bin/sh
set -eu
group=ff02::1:ff00:1
for vid in 110 120; do
  if ! bridge mdb replace dev br-bird port br-bird grp "$group" vid "$vid" 2>/dev/null; then
    bridge mdb del dev br-bird port br-bird grp "$group" vid "$vid" 2>/dev/null || true
    bridge mdb add dev br-bird port br-bird grp "$group" vid "$vid"
  fi
done
```

The script refreshes each entry with `bridge mdb replace`. Older iproute2 releases, including 6.1 in Debian 12 and Ubuntu 24.04, do not have that command, so the script deletes and re-adds the entry instead.

Both example IPv6 addresses map to the solicited-node group `ff02::1:ff00:1`, and so do the EUI-64 link-local addresses derived from the example MAC. A solicited-node group has the form `ff02::1:ffXX:XXXX`, where the last 24 bits come from the unicast address. Check the actual addresses with `ip -6 address show dev v110` and `ip -6 address show dev v120`. If you change the IPv6 addresses, the MAC address, or the address generation mode, recalculate the groups and add an entry for each distinct group.

Make the script executable, run it, and check the result:

```sh
chmod 0755 /usr/local/sbin/bird-nd-refresh
/usr/local/sbin/bird-nd-refresh
bridge mdb show dev br-bird
```

Use a systemd timer to run the script every 60 seconds, well within the 260-second expiry.

Create `/etc/systemd/system/bird-nd-refresh.service` with:

```ini
[Unit]
Description=Refresh BIRD example IPv6 neighbor multicast memberships
After=network-online.target
ConditionPathExists=/sys/class/net/br-bird

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/bird-nd-refresh
```

Create `/etc/systemd/system/bird-nd-refresh.timer` with:

```ini
[Unit]
Description=Renew BIRD example IPv6 neighbor multicast memberships

[Timer]
OnBootSec=30s
OnUnitActiveSec=60s
AccuracySec=1s

[Install]
WantedBy=timers.target
```

Start the timer:

```sh
systemctl daemon-reload
systemctl enable --now bird-nd-refresh.timer
```

The timer maintains only the groups listed in the script. The kernel joins other groups as well, such as the all-routers group `ff02::2`, and those still expire. If you adapt the configuration, make sure that neighbor discovery, Router Advertisements, and any other IPv6 control traffic you rely on still reach the CPU.

### 5. Configure BIRD

Configure the peers to advertise these example loopback prefixes, and set AS 65000 as the remote AS on both:

| Peer | AS | IPv4 prefix | IPv6 prefix |
| --- | --- | --- | --- |
| Left | 65001 | `203.0.113.1/32` | `2001:db8:1::1/128` |
| Right | 65002 | `203.0.113.2/32` | `2001:db8:2::1/128` |

These documentation prefixes and private AS numbers are meant for an isolated test network. Do not announce them to the Internet.

Back up `/etc/bird/bird.conf` if it exists, then replace its contents with:

```conf
log syslog all;
router id 192.0.2.1;

protocol device {
  scan time 1;
}

protocol direct connected {
  ipv4;
  ipv6;
  interface "v110", "v120";
}

filter allowed_v4 {
  if net ~ [203.0.113.0/24{32,32}] then accept;
  reject;
}

filter allowed_v6 {
  if net ~ [2001:db8::/32{128,128}] then accept;
  reject;
}

protocol kernel kernel4 {
  kernel table 254;
  scan time 1;
  ipv4 { import none; export filter allowed_v4; };
}

protocol kernel kernel6 {
  kernel table 254;
  scan time 1;
  ipv6 { import none; export filter allowed_v6; };
}

template bgp peer4 {
  local as 65000;
  hold time 9;
  keepalive time 3;
  connect retry time 2;
  graceful restart off;
  ipv4 { import filter allowed_v4; export filter allowed_v4; };
}

template bgp peer6 {
  local as 65000;
  hold time 9;
  keepalive time 3;
  connect retry time 2;
  graceful restart off;
  ipv6 { import filter allowed_v6; export filter allowed_v6; };
}

protocol bgp left4 from peer4 {
  local 192.0.2.1;
  neighbor 192.0.2.2 as 65001;
}

protocol bgp right4 from peer4 {
  local 198.51.100.1;
  neighbor 198.51.100.2 as 65002;
}

protocol bgp left6 from peer6 {
  local 2001:db8:110::1;
  neighbor 2001:db8:110::2 as 65001;
}

protocol bgp right6 from peer6 {
  local 2001:db8:120::1;
  neighbor 2001:db8:120::2 as 65002;
}
```

BIRD runs as the `bird` user, so the file must stay readable by that user. The Debian and Ubuntu packages install it with this ownership and mode:

```sh
chown bird:bird /etc/bird/bird.conf
chmod 0640 /etc/bird/bird.conf
```

The configuration works as follows:

- The Device and Direct protocols give BIRD interface and connected-route information.
- The Kernel protocols export the permitted routes into the Linux main table (254) and do not import unrelated kernel routes into BIRD. The driver offloads routes only from the main table; routes in other tables are forwarded by the CPU.
- The import and export filters accept only the example host prefixes, so default routes and management prefixes are rejected.
- Each peer and address family has its own BGP session, four in total.
- The short timers and disabled graceful restart make route withdrawal and recovery easy to observe. Choose timers and a restart policy that suit your deployment.

Validate the configuration, then start BIRD:

```sh
bird -p -c /etc/bird/bird.conf
systemctl start bird.service
systemctl status bird.service
```

For later changes, validate first and then reload:

```sh
bird -p -c /etc/bird/bird.conf && birdc configure
```

### 6. Verify Sessions, Routes, and Forwarding

Check the control plane:

```sh
birdc show protocols
birdc show route all
```

Once the peers are configured and connected, `left4`, `right4`, `left6`, and `right6` should reach the `Established` state, and BIRD should select the loopback routes the peers advertise.

Then check the Linux routes, neighbors, and VLAN membership:

```sh
ip -d route show table main
ip -6 -d route show table main
ip -j route show exact 203.0.113.2/32
ip -6 -j route show exact 2001:db8:2::1/128
ip -s neigh show
bridge vlan show
bridge mdb show dev br-bird
```

Linux should route the right-side destinations through `198.51.100.2` and `2001:db8:120::2` on `v120`, and the left-side destinations through `v110`. Routes that the driver has offloaded carry the `rt_offload` flag. IPv6 routes should use the peers' global addresses as nexthops, as here, not `fe80::` addresses: the driver ignores link-local neighbor entries, so traffic through a link-local nexthop can stay on the CPU path.

To test forwarding between the peers, use their loopbacks as source and destination. Run these commands on the left peer:

```sh
ping -I 203.0.113.1 -c 5 203.0.113.2
ping -6 -I 2001:db8:1::1 -c 5 2001:db8:2::1
```

Pinging from the Tactical 1000 itself does not test the same path, because those packets originate on the CPU.

A BGP session, an installed route, or an offload flag does not by itself prove that the ASIC is forwarding the traffic. To confirm it, capture the same flow at the receiving peer and on the Tactical 1000 while the ping runs:

```sh
# On the right peer (replace eth1 with the interface facing the Tactical 1000)
tcpdump -eni eth1 -v 'host 203.0.113.1 or host 2001:db8:1::1'

# On the Tactical 1000
tcpdump -ni v110 'host 203.0.113.1 or host 2001:db8:1::1'
```

At the receiving peer, the packets should carry the Tactical 1000's router MAC as their source address and a TTL or hop limit one lower than the sender used. On the Tactical 1000, the capture should stay empty, because packets forwarded in hardware never reach the CPU.

An empty capture only counts if the capture itself works. Check it by pinging a Tactical 1000 address from the same source, for example with `ping -I 203.0.113.1 -c 5 192.0.2.1` on the left peer; these packets must appear. When you stop tcpdump, also make sure it reports no packets dropped by the kernel.

### 7. Make the Configuration Persistent

The `ip` and `bridge` commands above change only the running system. To recreate the bridge, port membership, VLAN interfaces, and addresses at boot, use your existing network manager or a boot-time setup service, and keep the MAC address update order from step 3. Do not let two network managers configure the same ports.

BIRD must start only after those interfaces and the neighbor discovery setup from step 4 are in place. If a custom service sets up the network, make `bird.service` require that service and start after it, for example through a drop-in created with `systemctl edit bird.service`. Do the same for `bird-nd-refresh.service` if you use it. If a network manager configures the interfaces instead, use its hooks to enforce the same order. Note that `network-online.target` alone does not guarantee that these VLAN interfaces are ready.

The sysctl file already persists across reboots, and the refresh timer, if you use it, also starts at boot. The querier settings are runtime settings like the bridge itself, so your boot-time network setup must apply them after it adds the VLANs. Once network startup and service ordering are in place, enable BIRD:

```sh
systemctl enable bird.service
```

After a planned reboot, repeat the session, route, neighbor, and peer-to-peer forwarding checks. Also check the neighbor discovery setup: `bridge vlan global show dev br-bird` for the querier, or the refresh timer:

```sh
systemctl status bird-nd-refresh.timer
journalctl -u bird.service -u bird-nd-refresh.service -b
```

## Performance

With hardware offload, routed TCP goodput over 10 GbE links can reach about 9.4 Gbit/s for IPv4 and 9.3 Gbit/s for IPv6. CPU load during these transfers stays around 7–10%, measured across the whole system and including monitoring overhead, not just for the BIRD process.

These are reference figures, not guaranteed capacity. Actual throughput and CPU load depend on packet sizes, traffic patterns, the endpoints, and kernel and driver support.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| BGP stays in `Active` or `Connect` | Peer addresses and AS numbers, physical link, VLAN membership, neighbor resolution, and whether the peer can reach TCP port 179 on the router |
| BIRD has a route but Linux does not | Whether the route is selected, Kernel protocol state, the export filter, and the kernel table number |
| IPv4 reaches the peer but is not offloaded | The running kernel's multipath option and Layer 3 driver support; a working BGP session does not imply offload |
| Route has an offload flag but traffic fails | Bridge MAC update order, next-hop MAC, neighbor state, and membership in both VLANs |
| IPv6 works at first, then fails | Whether the per-VLAN querier is enabled or the refresh timer is running, the solicited-node groups, and the current IPv6 addresses |
| Configuration disappears after a reboot | Network setup at boot and BIRD startup ordering; enabling `bird.service` alone does not restore the interfaces |

BIRD filters control which routes are exchanged, not which packets are allowed through. Separating traffic into VLANs at Layer 2 does not stop routing between them. Do not rely on Linux firewall rules, or on disabling software forwarding, to stop traffic that the ASIC forwards; verify hardware enforcement separately.

## Documentation

- [Official BIRD Documentation](https://bird.network.cz/?get_doc&f=bird.html&v=20)
- [Initial Network Configuration Guide](initial-network-configuration-guide.md)
- [Linux-Powered VLAN Segmentation](linux-powered-vlan-segmentation.md)
